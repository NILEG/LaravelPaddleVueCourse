```jsx
interface Plan {
    id: string;
    name: string;
    description: string;
    monthlyPrice: number;
    monthlyPriceID: string;
    yearlyPriceID: string;
    yearlyPrice: number;
    icon: any; // Component type
    popular: boolean;
    features: string[];
}

// Pricing plans
const plans: Plan[] = [
    {
        id: 'basic',
        name: 'Basic Plan',
        description: 'Perfect for individuals getting started',
        monthlyPriceID: 'pri_01jz0gj3ranjxc1smhn4pspdb7',
        yearlyPriceID: 'pri_01jz0gkvr89mccrb0qkney4x17',
        monthlyPrice: 10,
        yearlyPrice: 108,
        icon: Star,
        popular: false,
        features: ['Access to all basic Blogs', 'Mobile app access'],
    },
    {
        id: 'premium',
        name: 'Premium Plan',
        description: 'Best for professionals and growing teams',
        monthlyPrice: 15,
        monthlyPriceID: 'pri_01jz0gs9es47831z6jnz5tx989',
        yearlyPriceID: 'pri_01jz0gtpe7hw4mta0mthykayba',
        yearlyPrice: 144,
        icon: Zap,
        popular: true,
        features: ['Access to all Blogs', 'Mobile app access'],
    },
];
```
1. Import paddle sdk

```jsx
import { initializePaddle, Paddle } from '@paddle/paddle-js';
```

1. Import onMounted

```jsx
import { computed, ref, **onMounted** } from 'vue';
```

1. Initialize Paddle SDK

```jsx
// Initialize Paddle instance
const paddle = ref<Paddle | null>(null);

// Initialize Paddle on component mount
onMounted(async () => {
    try {
        const paddleInstance = await initializePaddle({ 
            environment: 'sandbox', // Change to 'production' for live environment
            token: '' // Replace with your actual auth token
        });
        
        if (paddleInstance) {
            paddle.value = paddleInstance;
        }
    } catch (error) {
        console.error('Failed to initialize Paddle:', error);
    }
});
```

1. Update the subscribe Button

```jsx
const subscribe = (planId: string, isYearlyPlan: boolean) => {
    try {
        const plan = plans.find((p) => p.id === planId);
        if (!plan) {
            console.error('Plan not found:', planId);
            return;
        }

        // Check if Paddle is initialized
        if (!paddle.value) {
            console.error('Paddle is not initialized yet');
            alert('Payment system is still loading. Please try again in a moment.');
            return;
        }

        console.log(`Subscribing to ${plan.name}`);
        console.log(`Billing: ${isYearlyPlan ? 'Yearly' : 'Monthly'}`);
        console.log(`Price: $${getPrice(plan)}`);
        
        // Get the correct price ID based on billing cycle
        const priceId = isYearlyPlan ? plan.yearlyPriceID : plan.monthlyPriceID;
        
        // Open Paddle checkout
        paddle.value.Checkout.open({
            items: [
                {
                    priceId: priceId,
                    quantity: 1
                }
            ],
            settings: {
                displayMode: "overlay",
            }
        });
        
    } catch (error) {
        console.error('Error subscribing:', error);
        alert('An error occurred while processing your subscription. Please try again.');
    }
};
```
1. Pass Custom Data in the checkout

```jsx
import { Head, **usePage** } from '@inertiajs/vue3';
const page = usePage();
const user = computed(() => page.props.auth.user);

paddle.value.Checkout.open({
            items: [
                {
                    priceId: priceId,
                    quantity: 1,
                },
            ],
            settings: {
                displayMode: 'overlay',
            },
            customData: {
                email: user.value?.email || '',
                userId: user.value?.id || '',
            },
        });
```
