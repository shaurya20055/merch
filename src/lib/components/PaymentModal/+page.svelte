<!-- src/lib/components/PaymentModal/+page.svelte -->

<script lang="ts">
  // @ts-nocheck
  let { product, onClose } = $props();
  
  let formData = $state({
    name: '',
    email: '',
    phone: '',
    address: '',
    size: 'M',
    paymentScreenshot: null,
    // Add state for the new checkboxes, defaulting to true
    includeNitsLogo: true,
    includeMlClubLogo: true
  });

  let isSubmitting = $state(false);
  let submitSuccess = $state(false);
  let errorMessage = $state<string | null>(null);

  function handleFileChange(event) {
    const file = event.target.files[0];
    if (file) {
      formData.paymentScreenshot = file;
    }
  }

  async function handleSubmit(event) {
    event.preventDefault();

    if (!formData.paymentScreenshot) {
      errorMessage = 'Please select a payment screenshot to upload.';
      return;
    }

    isSubmitting = true;
    errorMessage = null;

    try {
      // 1. Upload the screenshot
      const uploadFormData = new FormData();
      uploadFormData.append('screenshot', formData.paymentScreenshot);

      const uploadResponse = await fetch('/api/uploads', {
        method: 'POST',
        body: uploadFormData,
      });

      if (!uploadResponse.ok) {
        throw new Error('Screenshot upload failed. Please try again.');
      }
      
      const uploadData = await uploadResponse.json();
      const screenshotUrl = uploadData.screenshotUrl;

      // 2. Save the complete order to Supabase, now including logo choices
      const orderDetails = {
        product,
        screenshotUrl,
        fullName: formData.name,
        email: formData.email,
        phoneNumber: formData.phone,
        tshirtSize: formData.size,
        deliveryAddress: formData.address,
        // Pass the checkbox values to the API
        include_nits_logo: formData.includeNitsLogo,
        include_ml_club_logo: formData.includeMlClubLogo
      };

      const orderResponse = await fetch('/api/orders', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(orderDetails)
      });

      if(!orderResponse.ok) {
          const errorData = await orderResponse.json();
          throw new Error(errorData.details || 'Could not save your order. Please contact support.');
      }

      submitSuccess = true;
      
      setTimeout(() => {
        onClose();
      }, 3000);

    } catch (err: any) {
      errorMessage = err.message;
    } finally {
      isSubmitting = false;
    }
  }

  function handleBackdropClick(event) {
    if (event.target === event.currentTarget) {
      onClose();
    }
  }
  
  function handleKeyDown(event) {
    if (event.key === 'Escape') {
      onClose();
    }
  }
</script>

<div 
  class="fixed inset-0 bg-black/80 flex items-center justify-center p-4 z-50"
  onclick={handleBackdropClick}
  tabindex="0"
  onkeydown={handleKeyDown}
  role="dialog"
  aria-modal="true"
  aria-labelledby="modal-title"
>
  <div class="bg-card text-card-foreground rounded-2xl max-w-2xl w-full max-h-[90vh] overflow-y-auto">
    <div class="p-6">
      <div class="flex items-center justify-between mb-6">
        <h2 id="modal-title" class="font-sans font-bold text-2xl">Complete Your Order</h2>
        <button 
          onclick={onClose}
          class="text-muted-foreground hover:text-card-foreground transition-colors"
          aria-label="Close"
        >
          <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
          </svg>
        </button>
      </div>

      {#if submitSuccess}
        <div class="text-center py-8">
          <div class="w-16 h-16 bg-green-100 rounded-full flex items-center justify-center mx-auto mb-4">
            <svg class="w-8 h-8 text-green-600" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 13l4 4L19 7" />
            </svg>
          </div>
          <h3 class="font-sans font-bold text-xl mb-2">Order Submitted!</h3>
          <p class="text-muted-foreground">We'll verify your payment and contact you soon.</p>
        </div>
      {:else}
        <form onsubmit={handleSubmit} class="space-y-6">
          <!-- Personal Details -->
          <div class="space-y-4">
            <h3 class="font-sans font-semibold text-lg">Personal Details</h3>
            
            <div class="grid md:grid-cols-2 gap-4">
              <div>
                <label for="name" class="block text-sm font-medium mb-2">Full Name *</label>
                <input type="text" bind:value={formData.name} required class="w-full bg-input text-foreground px-4 py-3 rounded-lg border border-border focus:border-ring focus:outline-none" placeholder="Enter your full name" id="name" />
              </div>
              
              <div>
                <label for="email" class="block text-sm font-medium mb-2">Email *</label>
                <input type="email" bind:value={formData.email} required class="w-full bg-input text-foreground px-4 py-3 rounded-lg border border-border focus:border-ring focus:outline-none" placeholder="your.email@example.com" id="email" />
              </div>
            </div>

            <div class="grid md:grid-cols-2 gap-4">
              <div>
                <label for="phone" class="block text-sm font-medium mb-2">Phone Number *</label>
                <input type="tel" bind:value={formData.phone} required class="w-full bg-input text-foreground px-4 py-3 rounded-lg border border-border focus:border-ring focus:outline-none" placeholder="+91 987XXXXXXX" id="phone" />
              </div>
              
              <div>
                <label for="size" class="block text-sm font-medium mb-2">T-Shirt Size *</label>
                <select bind:value={formData.size} class="w-full bg-input text-foreground px-4 py-3 rounded-lg border border-border focus:border-ring focus:outline-none" id="size">
                  <option value="S">Small (S)</option>
                  <option value="M">Medium (M)</option>
                  <option value="L">Large (L)</option>
                  <option value="XL">Extra Large (XL)</option>
                  <option value="XXL">Double XL (XXL)</option>
                </select>
              </div>
            </div>

            <div>
              <label for="address" class="block text-sm font-medium mb-2">Delivery Address *</label>
              <textarea bind:value={formData.address} required rows="3" class="w-full bg-input text-foreground px-4 py-3 rounded-lg border border-border focus:border-ring focus:outline-none resize-none" placeholder="Enter your complete delivery address" id="address"></textarea>
            </div>
          </div>

          <!-- NEW Customization Section -->
          <div class="space-y-4">
            <h3 class="font-sans font-semibold text-lg">Customization</h3>
            <div class="grid sm:grid-cols-2 gap-4">
              <label class="flex items-center gap-3 bg-input p-4 rounded-lg border border-border has-[:checked]:border-primary has-[:checked]:bg-primary/10 transition-all cursor-pointer">
                <input type="checkbox" bind:checked={formData.includeNitsLogo} class="w-5 h-5 accent-primary" />
                <span>Include <strong>NIT Silchar</strong> Logo</span>
              </label>
              <label class="flex items-center gap-3 bg-input p-4 rounded-lg border border-border has-[:checked]:border-primary has-[:checked]:bg-primary/10 transition-all cursor-pointer">
                <input type="checkbox" bind:checked={formData.includeMlClubLogo} class="w-5 h-5 accent-primary" />
                <span>Include <strong>ML Club</strong> Logo</span>
              </label>
            </div>
          </div>

          <!-- Payment Section -->
          <div class="space-y-4">
            <h3 class="font-sans font-semibold text-lg">Payment Details</h3>
            
            <div class="bg-primary/10 border border-primary/20 rounded-lg p-4">
              <div class="flex items-start gap-4">
                <div class="flex-1">
                  <h4 class="font-semibold mb-2">Scan QR Code to Pay</h4>
                  <p class="text-sm text-muted-foreground mb-3">Scan the QR code below with any UPI app and pay ₹{product.price}</p>
                  <div class="text-sm">
                    <p><strong>Amount:</strong> ₹{product.price}</p>
                    <p><strong>UPI ID:</strong> mlclub@nits</p>
                  </div>
                </div>
                
                <div class="bg-white p-3 rounded-lg">
                  <img src="https://api.qrserver.com/v1/create-qr-code/?size=120x120&data=upi://pay?pa=ml.club@nits&pn=Merch%20Shop&am={product.price}&cu=INR" alt="Payment QR Code" class="w-30 h-30" />
                </div>
              </div>
            </div>

            <div>
              <label for="payment-screenshot" class="block text-sm font-medium mb-2">Upload Payment Screenshot *</label>
              <div class="border-2 border-dashed border-border rounded-lg p-6 text-center">
                <input type="file" accept="image/*" onchange={handleFileChange} required class="hidden" id="payment-screenshot" />
                <label for="payment-screenshot" class="cursor-pointer">
                  <div class="space-y-2">
                    <svg class="w-8 h-8 text-muted-foreground mx-auto" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M7 16a4 4 0 01-.88-7.903A5 5 0 1115.9 6L16 6a5 5 0 011 9.9M15 13l-3-3m0 0l-3 3m3-3v12" />
                    </svg>
                    <div>
                      <p class="text-sm font-medium">Click to upload payment screenshot</p>
                      <p class="text-xs text-muted-foreground">PNG, JPG up to 10MB</p>
                    </div>
                  </div>
                </label>
                {#if formData.paymentScreenshot}
                  <p class="text-sm text-primary mt-2">✓ {formData.paymentScreenshot.name}</p>
                {/if}
              </div>
            </div>
          </div>

          {#if errorMessage}
            <p class="text-red-500 text-center mt-4">{errorMessage}</p>
          {/if}

          <button type="submit" disabled={isSubmitting} class="w-full bg-primary hover:bg-primary/90 disabled:bg-primary/50 text-primary-foreground font-sans font-semibold py-4 px-8 rounded-lg transition-colors flex items-center justify-center gap-2" aria-label="Submit Order">
            {#if isSubmitting}
              <svg class="w-5 h-5 animate-spin" fill="none" viewBox="0 0 24 24">
                <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
                <path class="opacity-75" fill="currentColor" d="m4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
              </svg>
              Submitting Order...
            {:else}
              Submit Order
            {/if}
          </button>
        </form>
      {/if}
    </div>
  </div>
</div>
