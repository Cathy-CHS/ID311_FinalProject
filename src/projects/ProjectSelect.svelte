<script>
    import { onMount, createEventDispatcher } from 'svelte';
    import {colors, numBarShow, startingPoint, layerWidth, lineWidth,  HeightBetLayer, text_end, BPMorigin, text_start} from '../Constants.svelte';
    import {timeCursorMake,  timeCursorMove, grid, layerColoring, layerdrawing, makeButton, makeLayerSp} from '../layers/LayerSettings.svelte';
    import { text } from 'svelte/internal';
    import { ref, child, get, set, getDatabase, onValue } from 'firebase/database';

    export let [width, height, database, projToSee, NumBar, user] = [400,300, {}, []];
    
    console.log(database)
    let projectToSee = projToSee;
    console.log(database[projectToSee].Layers)
    let layers = database[projectToSee].Layers;

    console.log(width, height, NumBar)

    //max number of bar in one display
    //const numBarShow = 3;

    let BPM = BPMorigin;

    let soundObject = [
        {
            Inst: "piano",
            Soundtrack: []
        },
        {
            Inst: "guitar",
            Soundtrack: []
        },
        {
            Inst: "base",
            Soundtrack: []
        },
        {
            Inst: "cymbal",
            Soundtrack: []
        },
        {
            Inst: "snare",
            Soundtrack: []
        }
    ];
    let instList = ["piano", "guitar", "base", "cymbal", "snare"];
    const pianoPitchList = ['C#3','D#3','F#3','G#3','A#3','C#4','D#4','F#4','G#4','A#4','C#5','D#5','F#5','G#5','A#5','C3','D3','E3','F3','G3','A3','B3','C4','D4','E4','F4','G4','A4','B4','C5','D5','E5','F5','G5','A5','B5','C6'];
    const guitarPitchList = ['C#3','D#3','F#3','G#3','A#3','C#4','C3','D3','E3','F3','G3','A3','B3','C4','D4'];

    const dispatch=createEventDispatcher();
    let absoluteRaw = 0;
    let absoluteTick = 0;
    const sketch = (p5) =>{
        let timeCursor
        //showLocation: location of displayed starting point, previous bar*256 + location in bar -1
        let showLocation = 0;
        //pointer: location of time cursor (also a tick)
        let pointer = 0;
        //4/4 => 60/(BPM/4)s = 1 bar time. 1 bar = 256 tick
        // 1 bar time / 256 = 1 tick time
        // inverse of 1 tick time = fr
        let frameRate = 1/(60/(BPM/4)/256)

        let isPlay = 0;

        let incrementMulti = 1;

        p5.preload = () => {
            loadSoundtrack(soundObject);
        }

        p5.setup = async ()=>{
            p5.noCursor()
            p5.createCanvas(width, height);
            p5.noStroke();
            p5.frameRate(frameRate);
            timeCursor = timeCursorMake(p5, height);
            makeButtons()
            makeLayerSps()
        }

        let showHeight = 0;
        p5.draw = ()=>{
            p5.clear();
            p5.background(p5.color(colors.back));
            popUp()
            grid(p5, height, showLocation)
            drawSettings ()
            p5.textFont('Pretendard Medium');
            p5.textSize(height/40);
            p5.textWrap(p5.CHAR)
            for (let i=0; i<Object.keys(database).length;i++){
                let elemHeight = showHeight+(i+1)*HeightBetLayer
                let tempLayers = database[i.toString()].Layers
                for (let j=0; j<tempLayers.length;j++){
                    layerdrawing(p5, elemHeight, tempLayers[j]);
                }
                p5.noStroke()
                p5.fill(p5.color(colors.default))
                p5.text(database[i].Title, startingPoint/2, elemHeight- height/40*3/2, text_end/3.7)
            }
            keyboardHandler()
            absoluteRaw = timeCursorMove(p5, timeCursor, pointer, absoluteRaw, NumBar)
            mouseHandler()
            timegoes();
        }

        p5.mouseWheel = (a)=>{
            if (p5.mouseX>startingPoint){
                showHeight+=((a.delta>0)? -1:1)*height/25
                updateWheelSps()
            }
        }

        let layerSps=[]
        function makeLayerSps(){
            for (let i=0; i<Object.keys(database).length;i++){
                layerSps.push(makeLayerSp(p5, toggleToProject, showHeight, i, i))
            }
        }

        let backButton, duplButton, bpmButton, playButton, newProjButton
        let followButtons = []
        function makeButtons(){
            backButton = makeButton(p5, 'Back', function(){dispatch('totToTitle')}, 0, 0)
            bpmButton = makeButton(p5, 'BPMIcon', BPMchanger, 1)

            duplButton = makeButton(p5, 'AddProject', dupNewProject, 4)
            duplButton.y = HeightBetLayer*(projectToSee+1)
            playButton = makeButton(p5, 'songPlay', function(){isPlay = !isPlay}, 5)
            playButton.y = HeightBetLayer*(projectToSee+1)
            followButtons.push(duplButton)
            followButtons.push(playButton)
            newProjButton = makeButton(p5, 'AddProject', makeNewProject, 9)
            newProjButton.y = height*12/13
        }

        function dupNewProject(){
            let index = Object.keys(database).length
            database[index] = JSON.parse(JSON.stringify(database[projectToSee]))
            if (database[index].Title.length<=(25-5)) database[index].Title +="-Copy" 
            layerSps.push(makeLayerSp(p5, toggleToProject, showHeight, index, index))
            const db = getDatabase();
            set(ref(db, `${index}`), database[projectToSee]);
            projectToSee = index
            updateWheelSps()
            console.log(database)
        }

        function makeNewProject(){
            let newProject = {
            Maker : user,
            Title: "New Project",
            Tag : [],
            Desc : "Write the description in 200 characters",
            NumBar : "4",
            NumOrbit : 0,
            Origin : null,
            NumReproduction : 0,
            Layers :[{Inst:"base", Amplitude: 1, points:[]}]
            }
            let index = Object.keys(database).length
            database[index] = newProject
            const db = getDatabase();
            set(ref(db, `${index}`), newProject);
            layerSps.push(makeLayerSp(p5, toggleToProject, showHeight, index, index))
            console.log(database)
        }
        
        let BPMindex = 0
        let BPMpup = 0;
        function BPMchanger(){
            const BPMmulti = [1, 1.25, 1.5, 1.75, 2, 2.25, 2.5]
            BPMindex = ((BPMindex>=(BPMmulti.length-1))? 0: BPMindex+1);
            BPM = BPMorigin*BPMmulti[BPMindex]
            frameRate = 1/(60/(BPM/4)/256)
            //p5.frameRate(frameRate);
            BPMpup = frameRate
            incrementMulti = BPMmulti[BPMindex]
            console.log(BPM)
        }

        function popUp(){
            if(BPMpup){
                let fieldColor = p5.color(colors.default)
                fieldColor.setAlpha(100*BPMpup/frameRate);
                p5.fill(fieldColor)
                p5.textFont('Pretendard Medium');
                p5.textAlign(p5.CENTER, p5.CENTER)
                p5.textSize(height/3);
                p5.text('BPM = '+ BPM,width/2, height/2 )
                BPMpup--
                p5.textAlign(p5.LEFT, p5.TOP)
            }
        }

        function updateWheelSps(){
            if (layerSps.length>0) for (let layerSp of layerSps) layerSp.udt(showHeight)
            for (let button of followButtons){
                button.y = showHeight + HeightBetLayer*(projectToSee+1)
            }
        }

        function toggleToProject(clickProject){
            if (clickProject == projectToSee){
                dispatch('project', false);
                dispatch('projectnum', clickProject)
                p5.remove();
            }
            projectToSee = clickProject
            updateWheelSps()
            layers = database[projectToSee].Layers;
            console.log(layers)
        }


        function drawSettings () {
            p5.fill('#f5fafa');
            p5.textFont('Pretendard Black');
            let width_ratio = width/1920;
            p5.noStroke();
            p5.textSize(width_ratio*60);
            p5.textWrap(p5.WORD);
            p5.text("Project select",text_start,height*0.2, text_end*0.5);
            
            p5.textFont('Pretendard Medium');
            p5.textSize(width_ratio*30);
            p5.text("Scroll and click",text_start,height*0.4, text_end*0.5);
        };

        function keyboardHandler(){
            //pause
            if (p5.kb.presses('space')) {isPlay = !isPlay;}
        }

        function mouseHandler(){
            //interaction section
            p5.noStroke()
            p5.blendMode(p5.HARD_LIGHT);

            p5.fill(colors.default);
            p5.ellipse(p5.mouseX, p5.mouseY, lineWidth*20);
            
            p5.blendMode(p5.BLEND);
        }


        function timegoes(){
            if(isPlay){
                let diffCnt = Math.ceil(incrementMulti)
                for (let layer of layers) {
                    if (layer.points) {
                        for (let point of layer.points) {
                            if (absoluteTick == (point.bar-1)*256+point.start) {
                                if (layer.Inst == 'piano') {
                                    const inst = instList.indexOf(layer.Inst);
                                    const pitchnum = pianoPitchList.indexOf(point.pitch);
                                    soundObject[inst].Soundtrack[pitchnum].play(0, 1, layer.Amplitude);
                                    soundObject[inst].Soundtrack[pitchnum].stop(point.duration/frameRate);
                                } else if (layer.Inst == 'guitar') {
                                    const inst = instList.indexOf(layer.Inst);
                                    const pitchnum = guitarPitchList.indexOf(point.pitch);
                                    soundObject[inst].Soundtrack[pitchnum].play(0, 1, layer.Amplitude*point.amp/100);
                                } else {
                                    const inst = instList.indexOf(layer.Inst);
                                    soundObject[inst].Soundtrack[0].play(0, 1, layer.Amplitude*point.amp/100);
                                }
                            }
                            if(diffCnt>=1){
                                console.log(diffCnt)
                                console.log(absoluteTick, absoluteRaw)
                                for (let i=0; i<diffCnt; i++){
                                    if (absoluteTick-i == (point.bar-1)*256+point.start) {
                                        if (layer.Inst == 'piano') {
                                            const inst = instList.indexOf(layer.Inst);
                                            const pitchnum = pianoPitchList.indexOf(point.pitch);
                                            soundObject[inst].Soundtrack[pitchnum].play(0, 1, layer.Amplitude);
                                            soundObject[inst].Soundtrack[pitchnum].stop(point.duration/frameRate);
                                        } else if (layer.Inst == 'guitar') {
                                            const inst = instList.indexOf(layer.Inst);
                                            const pitchnum = guitarPitchList.indexOf(point.pitch);
                                            soundObject[inst].Soundtrack[pitchnum].play(0, 1, layer.Amplitude*point.amp/100);
                                        } else {
                                            const inst = instList.indexOf(layer.Inst);
                                            soundObject[inst].Soundtrack[0].play(0, 1, layer.Amplitude*point.amp/100);
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
                absoluteRaw = absoluteRaw+incrementMulti
                absoluteTick = Math.floor(absoluteRaw)
            }

            if (absoluteRaw<=numBarShow/2*256){
                pointer = absoluteRaw
                showLocation = 0;
            } else if (absoluteRaw>(NumBar-numBarShow/2)*256){
                pointer = absoluteRaw - (NumBar-numBarShow)*256; 
                showLocation = (NumBar-numBarShow)*256;
                if (absoluteRaw>=NumBar*256){
                    absoluteRaw = 0;
                    absoluteTick = 0;
                }
            }
            else {
                pointer =numBarShow/2*256;
                showLocation =absoluteRaw - numBarShow/2*256}
        }
        //For decoding drag
        let xToTick  = (X) => (X-startingPoint)*numBarShow*256/layerWidth
        let tickToTime = (tick) => [Math.floor((tick+showLocation)/256)+1, Math.round((tick+showLocation)%256)]

        function loadSoundtrack(soundObject) {
            // piano
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Db3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Eb3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Gb3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Ab3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Bb3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Db4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Eb4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Gb4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Ab4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Bb4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Db5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Eb5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Gb5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Ab5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/Bb5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/C3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/D3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/E3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/F3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/G3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/A3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/B3.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/C4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/D4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/E4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/F4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/G4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/A4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/B4.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/C5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/D5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/E5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/F5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/G5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/A5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/B5.mp3'));
            soundObject[0].Soundtrack.push(p5.loadSound('assets/piano/C6.mp3'));
            // guitar
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/Cs3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/Ds3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/Fs3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/Gs3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/As3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/Cs4.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/C3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/D3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/E3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/F3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/G3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/A3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/B3.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/C4.mp3'));
            soundObject[1].Soundtrack.push(p5.loadSound('assets/guitar/D4.mp3'));
            //bass
            soundObject[2].Soundtrack.push(p5.loadSound('assets/drum/bass.wav'));
            //cymbal
            soundObject[3].Soundtrack.push(p5.loadSound('assets/drum/ride.wav'));
            //snare
            soundObject[4].Soundtrack.push(p5.loadSound('assets/drum/snare.wav'));
        }
    }

    let sketchId;
    onMount(function () {
    let myp5 = new p5(sketch, sketchId);
    });

</script>

<div {sketchId} />