<script>
    import {
        createConfiguration,
        getConfiguration,
        primaryConfigurationSection,
        saveConfiguration,
    } from '../utils/configuration'
    import ToastStore from '../stores/toast'
    import SplashStore from '../stores/splash';

    import Select from '../components/Input/Select.svelte'
    import {writable} from 'svelte/store'
    import {onMount} from 'svelte'
    import Accordion from '../components/Accordion.svelte'
    import ConfigList from '../components/Config/ConfigList.svelte'
    import OrangeButton from '../components/Input/Button/OrangeButton.svelte'

    import {writeText} from '@tauri-apps/api/clipboard';
    import SuspenseButton from "../components/Input/Button/SuspenseButton.svelte";
    import Suspense from "../components/Suspense.svelte";
    import {fade, fly} from 'svelte/transition';
    import Init from "../splashscreens/Init.svelte";
    import {getLocalStorageKey, setLocalStorageKey} from "../utils/storage";

    const formData = writable({
        driver_config: {
            left_enabled: true,
            right_enabled: true,
            communication_protocol: 0,
            device_driver: 0,
            encoding_protocol: 0,
        },
    });


    let configurationOptions = [];


    const onFormSubmit = async () => {
        try {
            const result = createConfiguration(configurationOptions);
            await saveConfiguration(result);
            ToastStore.addToast(ToastStore.severity.SUCCESS, 'Success saving configuration. Please restart SteamVR for the changes to take effect.');
        } catch (e) {
            console.trace(e);
            if (Array.isArray(e))
                e.forEach(v => ToastStore.addToast(ToastStore.severity.ERROR, v));
            else
                ToastStore.addToast(ToastStore.severity.ERROR, e);
        }
    }

    let loaded = false;
    let fromCache = false;
    onMount(async () => {
        try {
            ({configurationOptions, fromCache} = await getConfiguration());
            const driver_openglove = configurationOptions.driver_openglove.options;

            if(!(getLocalStorageKey('initialised') === 'true') && !driver_openglove.left_enabled && !driver_openglove.right_enabled) {
                SplashStore.addSplash(Init, {
                    onActivate: async () => {
                        setLocalStorageKey('initialised', true);
                        driver_openglove.left_enabled = true;
                        driver_openglove.right_enabled = true;
                        await onFormSubmit();
                        SplashStore.popSplash();
                    }
                });
            }

            if(!fromCache) ToastStore.addToast(ToastStore.severity.SUCCESS, 'Successfully loaded configuration');
            loaded = true;
        } catch (e) {
            console.trace(e)
            if (Array.isArray(e))
                e.forEach(v => ToastStore.addToast(ToastStore.severity.ERROR, v));
            else
                ToastStore.addToast(ToastStore.severity.ERROR, e);
        }
    });

    const copyConfigurationToClipboard = () => {
        try {
            const result = createConfiguration(configurationOptions);

            writeText(JSON.stringify(result));

            ToastStore.addToast(ToastStore.severity.SUCCESS, 'Successfully copied to clipboard.');
        } catch (e) {
            console.trace(e);

            ToastStore.addToast(ToastStore.severity.ERROR, 'Could not copy configuration to clipboard.');
        }
    }
</script>
<div class="flex-grow max-w-md my-10 w-full">
    <div class="w-full flex flex-col justify-center items-center">
        <div class="mx-10 w-full overflow-auto">
            <h2 class="mb-5 text-center text-3xl font-extrabold  ">
                Driver Configuration
            </h2>
            <div class="shadow rounded">
                <div class="px-4 py-5 space-y-6">
                    <Suspense suspense={!loaded}>
                        {#each Object.entries(configurationOptions) as [key, value], i}
                            <div>
                                <Accordion title={value.title}>
                                    {#if Array.isArray(value.options)}
                                        <Select
                                                onSelectItemChanged={selectedKey => {
                                        configurationOptions[primaryConfigurationSection].options[key] = selectedKey;
                                    }}
                                                options={Object.entries(value.options).map(([k, v]) => ({
                                        title: v.title,
                                        value: parseInt(k),
                                    }))}
                                                defaultValue={configurationOptions[primaryConfigurationSection].options[key]}
                                        />
                                        <ConfigList
                                                bind:configItems={value.options[configurationOptions[primaryConfigurationSection].options[key]].options}/>
                                    {:else}
                                        <ConfigList
                                                hiddenKeys={Object.keys(configurationOptions).map((k) => k)}
                                                bind:configItems={configurationOptions[key].options}
                                        />
                                    {/if}
                                </Accordion>
                            </div>

                        {/each}
                    </Suspense>

                </div>
                <div class="flex flex-row px-4">
                    <OrangeButton onClick={copyConfigurationToClipboard}>Copy Config to Clipboard</OrangeButton>
                    <div class="flex-grow"></div>
                    <SuspenseButton onClick={onFormSubmit}>Save</SuspenseButton>
                </div>
            </div>
        </div>
    </div>
</div>