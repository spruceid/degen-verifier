<script>
    import Navaid from "navaid";
    import { onDestroy } from "svelte";
    import Home from "../routes/Home.svelte";

    let Route,
        params = {},
        active;
    let uri = location.pathname;
    $: active = uri.split("/");

    const router = Navaid("/")
        .on("/", () => (Route = Home))
        .listen();

    onDestroy(router.unlisten);
</script>

<main
    class="flex h-full w-full justify-center"
>
    <svelte:component this={Route} {params} />
    <!--
    <img
        src="/effect.svg"
        alt="background effect"
        class="absolute bottom-0 left-0 z-0"
    / >
    -->
</main>

<style global lang="postcss">
    @tailwind base;
    @tailwind components;
    @tailwind utilities;

    main {
        background: linear-gradient(158.67deg, #020736 30.84%, #300E92 103.49%);
        color: white;
    }
</style>
