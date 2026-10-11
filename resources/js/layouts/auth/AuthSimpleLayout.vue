<script setup lang="ts">
import { Link, usePage } from '@inertiajs/vue3';
import AppLogoIcon from '@/components/AppLogoIcon.vue';
import { home } from '@/routes';

const page = usePage();
const name = page.props.name;

defineProps<{
    title?: string;
    description?: string;
}>();
</script>

<template>
    <div class="auth-page-bg flex min-h-svh flex-col items-center justify-center px-5 py-8">
        <div class="flex w-full max-w-[390px] flex-col gap-6">
            <Link
                :href="home()"
                class="flex items-center justify-center gap-2 text-base font-semibold text-slate-900 dark:text-white"
            >
                <span
                    class="flex size-10 items-center justify-center rounded-md bg-white shadow-sm ring-1 ring-slate-200 dark:bg-slate-950 dark:ring-slate-800"
                >
                    <AppLogoIcon
                        class="size-7 fill-current text-blue-600 dark:text-blue-400"
                    />
                </span>
                <span>{{ name }}</span>
            </Link>

            <div class="auth-panel w-full">
                <div class="px-8 pt-9 pb-4 text-center sm:px-10">
                    <h1 class="text-lg font-semibold leading-none text-slate-950 dark:text-white">
                        {{ title }}
                    </h1>
                    <p class="mt-2 text-sm text-slate-500 dark:text-slate-400">
                        {{ description }}
                    </p>
                </div>

                <div class="px-8 pt-2 pb-8 sm:px-10 sm:pb-10">
                    <slot />
                </div>
            </div>

            <div class="text-center text-xs font-medium text-slate-500 dark:text-slate-400">
                Secured environment health monitoring
            </div>
        </div>
    </div>
</template>
