<script lang="ts">
  import { onMount } from 'svelte';
  import { Link } from "svelte-navigator";
  import ChatBot from "../components/ChatBot.svelte";
  import { propertiesStore } from '../stores';
  
  // Subscribe to the properties store
  let featuredProperties = [];
  let loading = false;
  let error = null;
  
  // Function to fetch featured properties
  async function loadFeaturedProperties() {
    const result = await propertiesStore.fetchFeaturedProperties();
    
    // If there's no data yet, use placeholder data
    if (!result.success || !result.data || result.data.length === 0) {
      featuredProperties = [
        {
          id: "1",
          title: "Modern Apartment in Downtown",
          price: 350000,
          bedrooms: 2,
          bathrooms: 2,
          area: 1200,
          address: {
            street: "123 Main St",
            city: "New York",
            state: "NY",
            zip: "10001",
            country: "USA"
          },
          images: ["https://images.unsplash.com/photo-1522708323590-d24dbb6b0267?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=2340&q=80"]
        },
        {
          id: "2",
          title: "Luxury Villa with Pool",
          price: 1200000,
          bedrooms: 4,
          bathrooms: 3.5,
          area: 3500,
          address: {
            street: "456 Ocean Ave",
            city: "Miami",
            state: "FL",
            zip: "33139",
            country: "USA"
          },
          images: ["https://images.unsplash.com/photo-1613977257363-707ba9348227?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=2340&q=80"]
        },
        {
          id: "3",
          title: "Cozy Suburban Home",
          price: 450000,
          bedrooms: 3,
          bathrooms: 2,
          area: 1800,
          address: {
            street: "789 Maple Dr",
            city: "Chicago",
            state: "IL",
            zip: "60007",
            country: "USA"
          },
          images: ["https://images.unsplash.com/photo-1568605114967-8130f3a36994?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=2340&q=80"]
        }
      ];
    } else {
      featuredProperties = result.data;
    }
  }
  
  // Fetch featured properties on mount
  onMount(() => {
    loadFeaturedProperties();
    
    // Start realtime subscription
    propertiesStore.startRealtimeSubscription();
    
    // Clean up subscription on component destroy
    return () => {
      propertiesStore.stopRealtimeSubscription();
    };
  });
  
  // Helper function to get the first image or a placeholder
  function getFirstImage(property) {
    if (property.images && property.images.length > 0) {
      return property.images[0];
    }
    return 'https://via.placeholder.com/800x600?text=No+Image+Available';
  }
</script>

<div class="bg-white">
  <!-- Hero Section -->
  <div class="relative">
    <div class="absolute inset-0">
      <img class="w-full h-full object-cover" src="https://images.unsplash.com/photo-1560518883-ce09059eeffa?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=2673&q=80" alt="Real Estate" />
      <div class="absolute inset-0 bg-gray-900 opacity-60"></div>
    </div>
    <div class="relative max-w-7xl mx-auto py-24 px-4 sm:py-32 sm:px-6 lg:px-8">
      <h1 class="text-4xl font-extrabold tracking-tight text-white sm:text-5xl lg:text-6xl">Find Your Dream Home</h1>
      <p class="mt-6 text-xl text-white max-w-3xl">
        Browse thousands of properties and connect with property owners directly. Our AI assistant is here to help you every step of the way.
      </p>
      <div class="mt-10 flex flex-col sm:flex-row gap-4">
        <Link to="/properties" class="inline-flex items-center justify-center px-5 py-3 border border-transparent text-base font-medium rounded-md text-white bg-primary-600 hover:bg-primary-700">
          Browse Properties
        </Link>
        <Link to="/register" class="inline-flex items-center justify-center px-5 py-3 border border-white text-base font-medium rounded-md text-white hover:bg-white hover:text-gray-900">
          List Your Property
        </Link>
      </div>
    </div>
  </div>

  <!-- Featured Properties Section -->
  <div class="max-w-7xl mx-auto py-16 px-4 sm:py-24 sm:px-6 lg:px-8">
    <div class="text-center">
      <h2 class="text-3xl font-extrabold text-gray-900 sm:text-4xl">Featured Properties</h2>
      <p class="mt-4 text-lg text-gray-500 max-w-2xl mx-auto">
        Explore our handpicked selection of premium properties from around the country.
      </p>
    </div>
    
    {#if loading}
      <div class="flex justify-center items-center py-12">
        <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-primary-600"></div>
      </div>
    {:else if error}
      <div class="text-center py-12">
        <p class="text-red-600">{error}</p>
        <button 
          class="mt-4 px-4 py-2 bg-primary-600 text-white rounded-md hover:bg-primary-700"
          on:click={() => propertiesStore.fetchFeaturedProperties()}
        >
          Try Again
        </button>
      </div>
    {:else}
      <div class="mt-12 grid gap-8 md:grid-cols-2 lg:grid-cols-3">
        {#each featuredProperties as property}
          <div class="bg-white overflow-hidden shadow-lg rounded-lg">
            <div class="h-48 w-full relative">
              <img src={getFirstImage(property)} alt={property.title} class="w-full h-full object-cover" />
            </div>
            <div class="p-6">
              <h3 class="text-lg font-semibold text-gray-900">{property.title}</h3>
              <p class="text-primary-600 text-xl font-bold mt-2">${property.price.toLocaleString()}</p>
              <div class="mt-4 flex items-center text-sm text-gray-500">
                <span class="mr-4">{property.bedrooms} Beds</span>
                <span class="mr-4">{property.bathrooms} Baths</span>
                <span>{property.area} sq ft</span>
              </div>
              <p class="mt-2 text-sm text-gray-500">{property.address.city}, {property.address.state}</p>
              <div class="mt-6">
                <Link to={`/properties/${property.id}`} class="inline-flex items-center justify-center w-full px-4 py-2 border border-transparent text-sm font-medium rounded-md text-white bg-primary-600 hover:bg-primary-700">
                  View Details
                </Link>
              </div>
            </div>
          </div>
        {/each}
      </div>
    {/if}
    
    <div class="mt-12 text-center">
      <Link to="/properties" class="inline-flex items-center justify-center px-5 py-3 border border-transparent text-base font-medium rounded-md text-white bg-primary-600 hover:bg-primary-700">
        View All Properties
      </Link>
    </div>
  </div>

  <!-- AI Assistant Section -->
  <div class="bg-gray-50">
    <div class="max-w-7xl mx-auto py-16 px-4 sm:py-24 sm:px-6 lg:px-8">
      <div class="lg:grid lg:grid-cols-2 lg:gap-8 items-center">
        <div>
          <h2 class="text-3xl font-extrabold text-gray-900 sm:text-4xl">
            Meet Your AI Real Estate Assistant
          </h2>
          <p class="mt-4 text-lg text-gray-500">
            Our intelligent AI assistant is here to help you find the perfect property, answer your questions, and guide you through the process.
          </p>
          <div class="mt-6 space-y-4">
            <div class="flex">
              <div class="flex-shrink-0">
                <svg class="h-6 w-6 text-primary-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                </svg>
              </div>
              <div class="ml-3">
                <h3 class="text-lg font-medium text-gray-900">Personalized Recommendations</h3>
                <p class="mt-2 text-base text-gray-500">Get property recommendations based on your preferences and search history.</p>
              </div>
            </div>
            <div class="flex">
              <div class="flex-shrink-0">
                <svg class="h-6 w-6 text-primary-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                </svg>
              </div>
              <div class="ml-3">
                <h3 class="text-lg font-medium text-gray-900">Instant Answers</h3>
                <p class="mt-2 text-base text-gray-500">Get immediate answers to your questions about properties, neighborhoods, and more.</p>
              </div>
            </div>
            <div class="flex">
              <div class="flex-shrink-0">
                <svg class="h-6 w-6 text-primary-600" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                  <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
                </svg>
              </div>
              <div class="ml-3">
                <h3 class="text-lg font-medium text-gray-900">24/7 Availability</h3>
                <p class="mt-2 text-base text-gray-500">Our AI assistant is available around the clock to help you with your real estate needs.</p>
              </div>
            </div>
          </div>
        </div>
        <div class="mt-12 lg:mt-0">
          <div class="bg-white rounded-lg shadow-lg overflow-hidden">
            <div class="px-6 py-8">
              <h3 class="text-lg font-medium text-gray-900 mb-4">Chat with our AI Assistant</h3>
              <ChatBot />
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>