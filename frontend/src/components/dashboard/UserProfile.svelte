<script lang="ts">
  import { onMount } from 'svelte';
  import { authStore } from '../../stores';
  import { supabase } from '../../lib/supabase';
  
  // Form state
  let fullName = '';
  let email = '';
  let phone = '';
  let bio = '';
  let avatarUrl = '';
  let avatarFile = null;
  
  // UI state
  let loading = true;
  let submitting = false;
  let errorMessage = '';
  let successMessage = '';
  let uploadProgress = 0;
  
  // Load user profile on mount
  onMount(async () => {
    await loadUserProfile();
  });
  
  // Load user profile data
  async function loadUserProfile() {
    loading = true;
    errorMessage = '';
    
    try {
      const userId = $authStore.user?.id;
      
      if (!userId) {
        throw new Error('User not authenticated');
      }
      
      // Fetch user profile from Supabase
      const { data, error } = await supabase
        .from('profiles')
        .select('*')
        .eq('id', userId)
        .single();
      
      if (error) {
        throw new Error(error.message);
      }
      
      // Populate form fields
      if (data) {
        fullName = data.full_name || '';
        email = $authStore.user.email || '';
        phone = data.phone || '';
        bio = data.bio || '';
        avatarUrl = data.avatar_url || '';
      }
    } catch (err) {
      console.error('Error loading user profile:', err);
      errorMessage = 'Failed to load your profile. Please try again.';
    } finally {
      loading = false;
    }
  }
  
  // Handle avatar file selection
  function handleAvatarChange(event) {
    const files = event.target.files;
    
    if (files && files.length > 0) {
      avatarFile = files[0];
      
      // Create preview URL
      const reader = new FileReader();
      reader.onload = (e) => {
        avatarUrl = String(e.target.result);
      };
      reader.readAsDataURL(avatarFile);
    }
  }
  
  // Upload avatar to Supabase Storage
  async function uploadAvatar() {
    if (!avatarFile) return avatarUrl;
    
    try {
      const userId = $authStore.user?.id;
      const fileExt = avatarFile.name.split('.').pop();
      const fileName = `${userId}-${Math.random().toString(36).substring(2, 15)}.${fileExt}`;
      const filePath = `${userId}/${fileName}`;
      
      // Upload file to Supabase Storage
      const { data, error } = await supabase.storage
        .from('avatars')
        .upload(filePath, avatarFile, {
          cacheControl: '3600',
          upsert: false
        });
      
      if (error) {
        throw new Error(`Error uploading avatar: ${error.message}`);
      }
      
      // Get public URL
      const { data: urlData } = supabase.storage
        .from('avatars')
        .getPublicUrl(filePath);
      
      return urlData.publicUrl;
    } catch (err) {
      console.error('Error uploading avatar:', err);
      throw err;
    }
  }
  
  // Update user profile
  async function handleSubmit() {
    submitting = true;
    errorMessage = '';
    successMessage = '';
    uploadProgress = 0;
    
    try {
      const userId = $authStore.user?.id;
      
      if (!userId) {
        throw new Error('User not authenticated');
      }
      
      // Upload avatar if a new one was selected
      let newAvatarUrl = avatarUrl;
      if (avatarFile) {
        newAvatarUrl = await uploadAvatar();
      }
      
      // Update profile in Supabase
      const { error } = await supabase
        .from('profiles')
        .upsert({
          id: userId,
          full_name: fullName,
          phone,
          bio,
          avatar_url: newAvatarUrl,
          updated_at: new Date().toISOString()
        });
      
      if (error) {
        throw new Error(error.message);
      }
      
      // Update auth store
      authStore.updateProfile({
        full_name: fullName,
        avatar_url: newAvatarUrl
      });
      
      successMessage = 'Profile updated successfully!';
      avatarFile = null;
    } catch (err) {
      console.error('Error updating profile:', err);
      errorMessage = 'Failed to update profile. Please try again.';
    } finally {
      submitting = false;
    }
  }
</script>

<div>
  <h2 class="text-2xl font-bold text-gray-900 mb-6">Your Profile</h2>
  
  {#if loading}
    <div class="flex justify-center items-center py-12">
      <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-primary-600"></div>
    </div>
  {:else}
    {#if errorMessage}
      <div class="mb-6 bg-red-50 p-4 rounded-md">
        <div class="flex">
          <div class="flex-shrink-0">
            <svg class="h-5 w-5 text-red-400" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
              <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zM8.707 7.293a1 1 0 00-1.414 1.414L8.586 10l-1.293 1.293a1 1 0 101.414 1.414L10 11.414l1.293 1.293a1 1 0 001.414-1.414L11.414 10l1.293-1.293a1 1 0 00-1.414-1.414L10 8.586 8.707 7.293z" clip-rule="evenodd" />
            </svg>
          </div>
          <div class="ml-3">
            <p class="text-sm font-medium text-red-800">{errorMessage}</p>
          </div>
        </div>
      </div>
    {/if}
    
    {#if successMessage}
      <div class="mb-6 bg-green-50 p-4 rounded-md">
        <div class="flex">
          <div class="flex-shrink-0">
            <svg class="h-5 w-5 text-green-400" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
              <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
            </svg>
          </div>
          <div class="ml-3">
            <p class="text-sm font-medium text-green-800">{successMessage}</p>
          </div>
        </div>
      </div>
    {/if}
    
    <form on:submit|preventDefault={handleSubmit} class="space-y-8">
      <div class="space-y-6">
        <!-- Avatar -->
        <div>
          <label for="avatar-upload" class="block text-sm font-medium text-gray-700">Profile Photo</label>
          <div class="mt-2 flex items-center">
            <div class="relative">
              <img
                src={avatarUrl || 'https://via.placeholder.com/150?text=User'}
                alt="Profile"
                class="h-24 w-24 rounded-full object-cover"
              />
              <div class="absolute bottom-0 right-0 bg-white rounded-full p-1 shadow-md cursor-pointer">
                <svg class="h-5 w-5 text-gray-700" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
                  <path d="M13.586 3.586a2 2 0 112.828 2.828l-.793.793-2.828-2.828.793-.793zM11.379 5.793L3 14.172V17h2.828l8.38-8.379-2.83-2.828z" />
                </svg>
                <input
                  id="avatar-upload"
                  name="avatar"
                  type="file"
                  accept="image/*"
                  class="sr-only"
                  on:change={handleAvatarChange}
                />
              </div>
            </div>
            <div class="ml-5">
              <button
                type="button"
                class="bg-white py-2 px-3 border border-gray-300 rounded-md shadow-sm text-sm leading-4 font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500"
                on:click={() => document.getElementById('avatar-upload').click()}
              >
                Change
              </button>
              <p class="mt-1 text-xs text-gray-500">JPG, PNG, or GIF up to 2MB</p>
            </div>
          </div>
        </div>
        
        <!-- Basic Info -->
        <div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
          <div>
            <label for="full-name" class="block text-sm font-medium text-gray-700">
              Full Name
            </label>
            <div class="mt-1">
              <input
                id="full-name"
                type="text"
                bind:value={fullName}
                class="shadow-sm focus:ring-primary-500 focus:border-primary-500 block w-full sm:text-sm border-gray-300 rounded-md"
                placeholder="Your full name"
              />
            </div>
          </div>
          
          <div>
            <label for="email" class="block text-sm font-medium text-gray-700">
              Email
            </label>
            <div class="mt-1">
              <input
                id="email"
                type="email"
                value={email}
                disabled
                class="shadow-sm bg-gray-50 block w-full sm:text-sm border-gray-300 rounded-md cursor-not-allowed"
              />
              <p class="mt-1 text-xs text-gray-500">Email cannot be changed</p>
            </div>
          </div>
          
          <div>
            <label for="phone" class="block text-sm font-medium text-gray-700">
              Phone Number
            </label>
            <div class="mt-1">
              <input
                id="phone"
                type="tel"
                bind:value={phone}
                class="shadow-sm focus:ring-primary-500 focus:border-primary-500 block w-full sm:text-sm border-gray-300 rounded-md"
                placeholder="Your phone number"
              />
            </div>
          </div>
        </div>
        
        <!-- Bio -->
        <div>
          <label for="bio" class="block text-sm font-medium text-gray-700">
            Bio
          </label>
          <div class="mt-1">
            <textarea
              id="bio"
              rows="4"
              bind:value={bio}
              class="shadow-sm focus:ring-primary-500 focus:border-primary-500 block w-full sm:text-sm border-gray-300 rounded-md"
              placeholder="Tell us a bit about yourself"
            ></textarea>
          </div>
          <p class="mt-2 text-sm text-gray-500">
            Brief description for your profile. This will be displayed publicly.
          </p>
        </div>
      </div>
      
      <!-- Submit Button -->
      <div class="pt-5">
        <div class="flex justify-end">
          <button
            type="button"
            on:click={loadUserProfile}
            class="bg-white py-2 px-4 border border-gray-300 rounded-md shadow-sm text-sm font-medium text-gray-700 hover:bg-gray-50 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500"
          >
            Reset
          </button>
          <button
            type="submit"
            disabled={submitting}
            class="ml-3 inline-flex justify-center py-2 px-4 border border-transparent shadow-sm text-sm font-medium rounded-md text-white bg-primary-600 hover:bg-primary-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-primary-500 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {#if submitting}
              <svg class="animate-spin -ml-1 mr-3 h-5 w-5 text-white" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
              </svg>
              {#if uploadProgress > 0 && uploadProgress < 100}
                Uploading Image ({uploadProgress}%)...
              {:else}
                Saving...
              {/if}
            {:else}
              Save
            {/if}
          </button>
        </div>
      </div>
    </form>
  {/if}
</div>