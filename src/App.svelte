<script>
    import { fly, fade, blur, slide, scale } from 'svelte/transition';
	import { onMount } from 'svelte';
    import {width, height} from './Constants.svelte';
	import Layer from "./layers/Layer.svelte";
    import Project from "./projects/Project.svelte";
    import ProjectSelect from './projects/ProjectSelect.svelte';
    import Mainscreen from './Mainscreen.svelte';
    import data from './firebase';
    import { ref, child, get, set, getDatabase, onValue } from 'firebase/database';

    // GLOBALS
    let database=undefined;
    let projToSee = 0
    let project;
    let layers;
    let layerToSee = 1;
    let NumBar;

    function getData() {
        return new Promise((resolve, reject) => {
            const db = data;
            const dbRef = ref(db);
            get(dbRef).then((snapshot) => {
                const data = snapshot.val();
                 database = data;
                console.log(database);
                console.log('Database is loaded');
                resolve(data);
            });
        })
    }

    // Start here
    async function main(){
        await getData();
        console.log('start')
    }
    main();

    let user = "Anon";

    function initProj(projTo){
        projToSee = projTo
        project = database[projToSee]
        console.log(project)
        layers = project.Layers;
        layerToSee = 1;
        NumBar = project.NumBar;
    }
    let toggle= {toggleLayer : false, toggleProject: false, toggleMain: true};
    let dupProjectToggle = false
    function layerToggle(){
        dupProjectToggle = false
        toggle.toggleLayer = !toggle.toggleLayer}
    function projToggle(){
        dupProjectToggle = false
        toggle.toggleProject = !toggle.toggleProject}
    function switchtoProject(){
        toggle.toggleMain = !toggle.toggleMain}
    function layerDuplicate(){layers.push(JSON.parse(JSON.stringify(layers[layerToSee])))}
    const layerDelete = event => {
        let toDelete = event.detail
        layerToggle()
        layers.splice(toDelete, 1);
    }
    const layerSwitch = event => {layerToSee=event.detail}
    const projSwitch = event => {
        projToSee=event.detail
        initProj(projToSee)
    }
    const changeDescs = event => {
        project.Title = event.detail[0]
        project.Desc = event.detail[1]
    }

    function dupProjectInside(){
      let index = Object.keys(database).length
      database[index] = JSON.parse(JSON.stringify(database[projToSee]))
      database[index].Title +="-Copy" 
      initProj(index)
      dupProjectToggle = true
    }
</script>

{#if database}
    {#if toggle.toggleMain}
        <Mainscreen on:start = {switchtoProject} {width} {height} />
    {:else}
        {#if !(toggle.toggleProject)}
            <ProjectSelect on:project = {projToggle}
            on:projectnum ={projSwitch}
            {width} {height} {database} {projToSee} {NumBar} {user}/>
        {:else if toggle.toggleLayer}
            <div transition:fade>
                <Layer 
                on:layerToProject = {layerToggle}
                on:layerDup={layerDuplicate} 
                on:deleteLayer={layerDelete}
                {width} {height} {layers} {layerToSee} {projToSee} {NumBar}/>
            </div>
        {:else if dupProjectToggle}
            <div transition:fade>
                <Project on:layer = {layerToggle}
                on:projToTot = {projToggle}
                on:projectTexts = {changeDescs}
                on:layernum ={layerSwitch}
                on:projDup = {dupProjectInside}
                {width} {height} {project} {projToSee} {NumBar}/>
            </div>
        {:else if !(toggle.toggleLayer)}
            <div transition:fade>
                <Project on:layer = {layerToggle}
                on:projToTot = {projToggle}
                on:projectTexts = {changeDescs}
                on:layernum ={layerSwitch}
                on:projDup = {dupProjectInside}
                {width} {height} {project} {projToSee} {NumBar}/>
            </div>
        {/if}
    {/if}
{/if}