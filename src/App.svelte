<script>
    import AddNew from "./AddNew.svelte";
    import Mind3 from "./Mind3.svelte";
</script>

<main id="main" class="container">
    <Mind3 />
    <AddNew />
</main>

<style>
    .container {
        display: flex;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        height: 100vh;
        margin: 0;
        padding: 0;
    }
</style>
