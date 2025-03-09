<script lang="ts">
  import { onMount } from 'svelte';
  import { authStore } from '../../stores';
  import { supabase } from '../../lib/supabase';
  
  // State
  let conversations = [];
  let selectedConversation = null;
  let messages = [];
  let newMessage = '';
  let loading = true;
  let loadingMessages = false;
  let error = null;
  let conversationMap = new Map();
  
  // Fetch conversations on mount
  onMount(() => {
    fetchConversations();
    
    // Set up realtime subscription for new messages
    const messageSubscription = supabase
      .channel('public:messages')
      .on('postgres_changes', { 
        event: 'INSERT', 
        schema: 'public', 
        table: 'messages',
        filter: `receiver_id=eq.${$authStore.user?.id}`
      }, payload => {
        handleNewMessage(payload.new);
      })
      .subscribe();
    
    return () => {
      messageSubscription.unsubscribe();
    };
  });
  
  // Fetch user's conversations
  async function fetchConversations() {
    loading = true;
    error = null;
    
    try {
      const userId = $authStore.user?.id;
      
      if (!userId) {
        throw new Error('User not authenticated');
      }
      
      // Fetch conversations from Supabase
      const { data, error: fetchError } = await supabase
        .from('messages')
        .select(`
          id,
          sender_id,
          receiver_id,
          property_id,
          created_at,
          profiles:sender_id(id, full_name, avatar_url),
          properties:property_id(id, title, images)
        `)
        .or(`sender_id.eq.${userId},receiver_id.eq.${userId}`)
        .order('created_at', { ascending: false });
      
      if (fetchError) {
        throw new Error(fetchError.message);
      }
      
      // Group messages by conversation (unique combination of sender, receiver, and property)
      conversationMap = new Map();
      
      data.forEach(message => {
        // Determine the other party in the conversation
        const otherPartyId = message.sender_id === userId ? message.receiver_id : message.sender_id;
        const conversationKey = `${otherPartyId}-${message.property_id}`;
        
        if (!conversationMap.has(conversationKey)) {
          const otherParty = message.profiles;
          const property = message.properties;
          
          conversationMap.set(conversationKey, {
            id: conversationKey,
            otherPartyId,
            otherParty,
            property,
            lastMessage: message,
            unreadCount: message.sender_id !== userId && message.read === false ? 1 : 0
          });
        } else {
          const conversation = conversationMap.get(conversationKey);
          
          // Update unread count
          if (message.sender_id !== userId && message.read === false) {
            conversation.unreadCount += 1;
          }
          
          // Update last message if this one is newer
          if (new Date(message.created_at) > new Date(conversation.lastMessage.created_at)) {
            conversation.lastMessage = message;
          }
        }
      });
      
      sortConversations();
      
      // Select the first conversation if available
      if (conversations.length > 0 && !selectedConversation) {
        selectConversation(conversations[0]);
      }
    } catch (err) {
      console.error('Error fetching conversations:', err);
      error = 'Failed to load your messages. Please try again.';
    } finally {
      loading = false;
    }
  }
  
  // Select a conversation and load its messages
  async function selectConversation(conversation) {
    selectedConversation = conversation;
    loadingMessages = true;
    messages = [];
    
    try {
      const userId = $authStore.user?.id;
      
      // Fetch messages for this conversation
      const { data, error: fetchError } = await supabase
        .from('messages')
        .select('*')
        .or(`and(sender_id.eq.${userId},receiver_id.eq.${conversation.otherPartyId},property_id.eq.${conversation.property.id}),and(sender_id.eq.${conversation.otherPartyId},receiver_id.eq.${userId},property_id.eq.${conversation.property.id})`)
        .order('created_at', { ascending: true });
      
      if (fetchError) {
        throw new Error(fetchError.message);
      }
      
      messages = data;
      
      // Mark unread messages as read
      const unreadMessageIds = messages
        .filter(msg => msg.receiver_id === userId && !msg.read)
        .map(msg => msg.id);
      
      if (unreadMessageIds.length > 0) {
        await supabase
          .from('messages')
          .update({ read: true })
          .in('id', unreadMessageIds);
        
        // Update conversation unread count
        selectedConversation.unreadCount = 0;
        conversations = conversations.map(conv => 
          conv.id === selectedConversation.id ? selectedConversation : conv
        );
      }
    } catch (err) {
      console.error('Error fetching messages:', err);
      alert('Failed to load messages. Please try again.');
    } finally {
      loadingMessages = false;
    }
  }
  
  // Handle new message from subscription
  function handleNewMessage(message) {
    const userId = $authStore.user?.id;
    
    // Find the conversation this message belongs to
    const otherPartyId = message.sender_id === userId ? message.receiver_id : message.sender_id;
    const conversationKey = `${otherPartyId}-${message.property_id}`;
    const existingConversation = conversations.find(c => c.id === conversationKey);
    
    if (existingConversation) {
      // Update existing conversation
      if (message.sender_id !== userId) {
        existingConversation.unreadCount += 1;
      }
      
      existingConversation.lastMessage = message;
      
      // Move conversation to top
      conversations = [
        existingConversation,
        ...conversations.filter(c => c.id !== conversationKey)
      ];
      
      // Add message to current conversation if selected
      if (selectedConversation && selectedConversation.id === conversationKey) {
        messages = [...messages, message];
        
        // Mark as read if we're currently viewing this conversation
        if (message.sender_id !== userId) {
          supabase
            .from('messages')
            .update({ read: true })
            .eq('id', message.id);
          
          existingConversation.unreadCount = 0;
        }
      }
    } else {
      // We need to fetch the full conversation details
      fetchConversations();
    }
  }
  
  // Send a new message
  async function sendMessage() {
    if (!newMessage.trim() || !selectedConversation) return;
    
    try {
      const userId = $authStore.user?.id;
      
      // Create new message
      const { data, error: sendError } = await supabase
        .from('messages')
        .insert({
          sender_id: userId,
          receiver_id: selectedConversation.otherPartyId,
          property_id: selectedConversation.property.id,
          content: newMessage.trim(),
          read: false
        })
        .select();
      
      if (sendError) {
        throw new Error(sendError.message);
      }
      
      // Add message to the conversation
      if (data && data.length > 0) {
        messages = [...messages, data[0]];
        
        // Update last message in conversation
        selectedConversation.lastMessage = data[0];
        conversations = conversations.map(conv => 
          conv.id === selectedConversation.id ? selectedConversation : conv
        );
      }
      
      // Clear input
      newMessage = '';
    } catch (err) {
      console.error('Error sending message:', err);
      alert('Failed to send message. Please try again.');
    }
  }
  
  // Sort conversations by last message date
  function sortConversations() {
    if (conversationMap && conversationMap.size > 0) {
      conversations = Array.from(conversationMap.values())
        .sort((a, b) => new Date(b.lastMessage.created_at).getTime() - new Date(a.lastMessage.created_at).getTime());
    } else {
      conversations = [];
    }
  }
  
  // Format date for display
  function formatDate(dateString) {
    const date = new Date(dateString);
    const now = new Date();
    const diffDays = Math.floor((now.getTime() - date.getTime()) / (1000 * 60 * 60 * 24));
    
    if (diffDays === 0) {
      return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
    } else if (diffDays === 1) {
      return 'Yesterday';
    } else if (diffDays < 7) {
      return date.toLocaleDateString([], { weekday: 'long' });
    } else {
      return date.toLocaleDateString();
    }
  }
  
  // Get property image
  function getPropertyImage(property) {
    if (property.images && property.images.length > 0) {
      return property.images[0];
    }
    return 'https://via.placeholder.com/50x50?text=Property';
  }
  
  // Get avatar image
  function getAvatarImage(profile) {
    if (profile && profile.avatar_url) {
      return profile.avatar_url;
    }
    return 'https://via.placeholder.com/40x40?text=User';
  }
</script>

<div class="h-full flex flex-col">
  <h2 class="text-2xl font-bold text-gray-900 mb-6">Messages</h2>
  
  {#if loading}
    <div class="flex-grow flex justify-center items-center">
      <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-primary-600"></div>
    </div>
  {:else if error}
    <div class="bg-red-50 p-4 rounded-md mb-6">
      <div class="flex">
        <div class="flex-shrink-0">
          <svg class="h-5 w-5 text-red-400" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd" />
          </svg>
        </div>
        <div class="ml-3">
          <p class="text-sm font-medium text-red-800">{error}</p>
        </div>
      </div>
    </div>
  {:else if conversations.length === 0}
    <div class="bg-white shadow overflow-hidden sm:rounded-md p-6 text-center">
      <svg class="mx-auto h-12 w-12 text-gray-400" fill="none" viewBox="0 0 24 24" stroke="currentColor">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 10h.01M12 10h.01M16 10h.01M9 16H5a2 2 0 01-2-2V6a2 2 0 012-2h14a2 2 0 012 2v8a2 2 0 01-2 2h-5l-5 5v-5z" />
      </svg>
      <h3 class="mt-2 text-lg font-medium text-gray-900">No messages</h3>
      <p class="mt-1 text-sm text-gray-500">You don't have any messages yet. Contact property owners or wait for inquiries about your listings.</p>
    </div>
  {:else}
    <div class="flex-grow flex overflow-hidden bg-white shadow sm:rounded-lg">
      <!-- Conversation List -->
      <div class="w-1/3 border-r border-gray-200 overflow-y-auto">
        <ul class="divide-y divide-gray-200">
          {#each conversations as conversation}
            <li>
              <button
                on:click={() => selectConversation(conversation)}
                class="w-full px-4 py-4 flex items-start hover:bg-gray-50 focus:outline-none focus:bg-gray-50 {selectedConversation?.id === conversation.id ? 'bg-gray-50' : ''}"
              >
                <div class="flex-shrink-0 relative">
                  <img
                    src={getAvatarImage(conversation.otherParty)}
                    alt="User"
                    class="h-10 w-10 rounded-full"
                  />
                  {#if conversation.unreadCount > 0}
                    <span class="absolute -top-1 -right-1 bg-red-500 text-white text-xs rounded-full h-5 w-5 flex items-center justify-center">
                      {conversation.unreadCount}
                    </span>
                  {/if}
                </div>
                <div class="ml-3 flex-1 min-w-0">
                  <div class="flex justify-between">
                    <p class="text-sm font-medium text-gray-900 truncate">
                      {conversation.otherParty?.full_name || 'User'}
                    </p>
                    <p class="text-xs text-gray-500">
                      {formatDate(conversation.lastMessage.created_at)}
                    </p>
                  </div>
                  <div class="mt-1 flex items-center">
                    <div class="flex-shrink-0 h-8 w-8 mr-2">
                      <img
                        src={getPropertyImage(conversation.property)}
                        alt="Property"
                        class="h-full w-full rounded object-cover"
                      />
                    </div>
                    <p class="text-xs text-gray-500 truncate">
                      {conversation.property?.title || 'Property'}
                    </p>
                  </div>
                  <p class="mt-1 text-sm text-gray-500 truncate {conversation.unreadCount > 0 ? 'font-semibold' : ''}">
                    {conversation.lastMessage.content}
                  </p>
                </div>
              </button>
            </li>
          {/each}
        </ul>
      </div>
      
      <!-- Message Thread -->
      <div class="w-2/3 flex flex-col">
        {#if selectedConversation}
          <!-- Conversation Header -->
          <div class="px-4 py-3 border-b border-gray-200 flex items-center">
            <div class="flex-shrink-0">
              <img
                src={getAvatarImage(selectedConversation.otherParty)}
                alt="User"
                class="h-10 w-10 rounded-full"
              />
            </div>
            <div class="ml-3 min-w-0">
              <p class="text-sm font-medium text-gray-900">
                {selectedConversation.otherParty?.full_name || 'User'}
              </p>
              <div class="flex items-center mt-1">
                <div class="flex-shrink-0 h-5 w-5 mr-1">
                  <img
                    src={getPropertyImage(selectedConversation.property)}
                    alt="Property"
                    class="h-full w-full rounded object-cover"
                  />
                </div>
                <p class="text-xs text-gray-500 truncate">
                  {selectedConversation.property?.title || 'Property'}
                </p>
              </div>
            </div>
          </div>
          
          <!-- Messages -->
          <div class="flex-grow p-4 overflow-y-auto">
            {#if loadingMessages}
              <div class="flex justify-center items-center h-full">
                <div class="animate-spin rounded-full h-8 w-8 border-t-2 border-b-2 border-primary-600"></div>
              </div>
            {:else if messages.length === 0}
              <div class="flex justify-center items-center h-full text-gray-500">
                No messages yet. Start the conversation!
              </div>
            {:else}
              <div class="space-y-4">
                {#each messages as message}
                  <div class="flex {message.sender_id === $authStore.user?.id ? 'justify-end' : 'justify-start'}">
                    <div class="max-w-xs lg:max-w-md {message.sender_id === $authStore.user?.id ? 'bg-primary-100 text-primary-800' : 'bg-gray-100 text-gray-800'} rounded-lg px-4 py-2 shadow-sm">
                      <p class="text-sm">{message.content}</p>
                      <p class="text-xs text-gray-500 text-right mt-1">
                        {formatDate(message.created_at)}
                      </p>
                    </div>
                  </div>
                {/each}
              </div>
            {/if}
          </div>
          
          <!-- Message Input -->
          <div class="px-4 py-3 border-t border-gray-200">
            <form on:submit|preventDefault={sendMessage} class="flex">
              <input
                type="text"
                bind:value={newMessage}
                placeholder="Type a message..."
                class="flex-grow shadow-sm focus:ring-primary-500 focus:border-primary-500 block w-full sm:text-sm border-gray-300 rounded-md"
              />
              <button
                type="submit"
                disabled={!newMessage.trim()}
                class="ml-3 inline-flex items-center px-4 py-2 border border-transparent text-sm font-medium rounded-md shadow-sm text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 disabled:opacity-50 disabled:cursor-not-allowed"
              >
                Send
              </button>
            </form>
          </div>
        {:else}
          <div class="flex-grow flex justify-center items-center text-gray-500">
            Select a conversation to view messages
          </div>
        {/if}
      </div>
    </div>
  {/if}
</div>