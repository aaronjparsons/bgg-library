<script lang="ts">
    import type { ToastSettings } from '@skeletonlabs/skeleton';
    import { modalStore, toastStore } from '@skeletonlabs/skeleton';

    let submitting = false;

    const handleSubmit = async (event) => {
        submitting = true;
        const data = new FormData(event.target);

        const response = await fetch('https://formspree.io/f/xjvdnwzd', {
            method: 'POST',
            body: data,
            headers: {
                'Accept': 'application/json'
            }
        });

        if (response.ok) {
            const toast: ToastSettings = {
                message: 'Feedback successfully submitted. Thank you!',
                background: 'variant-filled-secondary',
                autohide: true,
                timeout: 5000
            };
            toastStore.trigger(toast);
            modalStore.close();
        } else {
            const toast: ToastSettings = {
                message: 'There was an error submitting your feedback. Please try again',
                background: 'variant-filled-primary',
                autohide: true,
                timeout: 5000
            };
            toastStore.trigger(toast);
        }
        submitting = false;
    }
</script>

<div class="card w-modal p-4 shadow-lg">
    <h1 class="h1 mb-6">Feedback</h1>
    <form on:submit|preventDefault={handleSubmit}>
        <textarea
            class="textarea"
            name="message"
            rows="4"
            placeholder="Feedback or suggestions?"
            required
        />
        <div class="flex justify-end mt-4">
            <button class="btn variant-ringed-surface mr-6" on:click|preventDefault={() => modalStore.close()}>
                Close
            </button>
            <button class="btn variant-filled-secondary" type="submit" disabled={submitting}>
                {submitting ? 'Submitting...' : 'Submit'}
            </button>
        </div>
    </form>
</div>