<template>
    <div  class="animatable_content_box ">
        <div class="content_toolbox" style="margin-bottom: -13px; ">
            
            <div v-if="inp_img && stable_diffusion.is_input_avail" class="l_button" style="float:right " @click="clear" > Reset</div>
            <div v-if="undo_history.length > 0  && stable_diffusion.is_input_avail" class="l_button" style="float:right " @click="do_undo" > Undo</div>
            <div v-if="inp_img " class="l_button" style="float:right " @click="save_img" > Save Image</div>
            <div v-if="inp_img " class="l_button" style="float:right " @click="is_invert = !is_invert" > Invert Mask</div>
            <div v-if="retry_params  && stable_diffusion.is_input_avail" class="l_button" style="float:right "   @click="generate(true)"> Retry</div>
            
            
            <div v-if="stable_diffusion.is_input_avail && inp_img"   style="float:right ; margin-right: 10px" >
                <input v-model="stroke_size_no" style="zoom:0.8; margin-top: 7px; width:100px" type="range"
                        min="1" max="100" >                
            </div>
            <div v-if="stable_diffusion.is_input_avail && inp_img"   class="l_button no_hover_bg" style="float:right ; margin-right: -5px; " > Stroke Size</div>
            

        </div>

        <div class="inapint_container" @drop.prevent="onDragFile" @dragover.prevent>
            <ImageCanvas style="cursor: crosshair;"  v-if="inp_img"  ref="inp_img_canvas" :is_inpaint="true" :is_invert="is_invert" :image_source="inp_img"  :is_disabled="!stable_diffusion.is_input_avail" id="inpaint"  canvas_id="inpaintcan" canvas_d_id="inpaintcand" :stroke_size_no="stroke_size_no" ></ImageCanvas>
            <div v-else @click="open_input_image" style=" "  :class="{ pointer_cursor  : is_sd_active }" >
                <center>
                    <p style="padding-top: calc( 50vh - 115px);padding-bottom: calc( 50vh - 155px);  opacity: 70%;" >Click to add input image and draw a mask</p>
                </center>
            </div>
        </div>

        <div class="bottom_toolbox"> 
            <textarea 
                    v-model="prompt" 
                    placeholder="Enter your prompt here" 
                    style="border-radius: 12px 12px 12px 12px; resize: none; " 
                    class="form-control inpaint_textbox"  
                    v-bind:class="{ 'disabled' : !stable_diffusion.is_input_avail}"
                    :rows="1">

                    <!-- v-bind:class="{ 'disabled' : !stable_diffusion.is_input_avail}" -->
            </textarea>

            <div  v-if="stable_diffusion.is_input_avail" @click="generate()"  class="l_button button_medium button_colored" style="float:right ; margin-top: 4px; ">Inpaint</div>
            <span v-else-if="stable_diffusion.generated_by=='inpainting'"   >
                <div v-if="is_stopping" class="l_button button_medium button_colored" style="float:right ; margin-top: 4px; " @click="stop_generation">Stopping</div>
                <div v-else class="l_button button_medium button_colored" style="float:right ; margin-top: 4px; " @click="stop_generation">Stop</div>
            </span>

        </div>

        <div v-if="backend_error" style="color:red ; margin-top:50px;">
            <div class="center loader_box">
                <p>Error: {{backend_error}} </p>
            </div>
        </div>

        <div v-if="!stable_diffusion.is_input_avail && stable_diffusion.generated_by=='inpainting'">
            <LoaderModal :loading_percentage="done_percentage" loading_title="Generating" :loading_desc="stable_diffusion.generation_state_msg" :remaining_times="stable_diffusion.remaining_times"></LoaderModal>
        </div>


    </div>
   
   
     
</template>
<script>

import ImageCanvas from '../components_bare/ImageCanvas.vue'
import LoaderModal from '../components_bare/LoaderModal.vue'
import Vue from 'vue'


export default {
    name: 'Inpainting',
    props: {
        app_state : Object   , 
        stable_diffusion : Object,
    },
    components: {LoaderModal ,ImageCanvas  },
    mounted() {

    },
    computed:{
        is_sd_active(){
            return this.stable_diffusion.is_input_avail;
        }
    },
    data() {
        return {
            prompt: "",
            inp_img : "",
            input_img_orig : "", // the first input image after reset
            backend_error : "",
            done_percentage : -1,
            is_stopping : false,
            undo_history : [],
            retry_params: undefined,
            stroke_size_no : "30",
            is_invert: false,
            "history_key" : "",
        };
    },
    methods: {
        
         generate(is_retry){

            let seed = 0;
            if(this.seed)
                seed = Number(this.seed);
            else
                seed = Math.floor(Math.random() * 100000);

            let prompt ;
            let input_image;
            let mask_img ;

            if(is_retry && this.retry_params){
                prompt = this.retry_params.prompt;
                input_image = this.retry_params.input_image;
                mask_img = this.retry_params.mask_img;
            }
            else{
                if(this.prompt.trim() == ""){
                    Vue.$toast.default('You need to enter a prompt')
                    return;
                }
                

                if(!this.inp_img){
                    Vue.$toast.default('You need to add an input image')
                    return;
                }
                    

                if(!this.$refs.inp_img_canvas.is_something_drawn){
                    Vue.$toast.default('You need to draw a mask')
                    return;
                }
                   

                prompt = this.prompt;

                input_image = window.ipcRenderer.sendSync('save_b64_image',  this.$refs.inp_img_canvas.get_img_b64(), true  );
                mask_img = window.ipcRenderer.sendSync('save_b64_image',  this.$refs.inp_img_canvas.get_mask_b46() , true );
            }

            this.retry_params = {input_image:input_image , mask_img:mask_img , prompt:prompt  }
            
            let params = {
                prompt : prompt , 
                W : -1 , 
                H : -1, 
                seed :seed,
                scale : this.guidence_scale , 
                input_image : input_image,
                is_inpaint :  true ,
                mask_image: mask_img , 
                model_id: 1 , 
            }

            let that = this;
            this.backend_error = "";
            this.done_percentage = -1;

            this.undo_history.push(input_image)

            let callbacks = {
                on_img(img_path){
                    that.inp_img = img_path
                    that.$refs.inp_img_canvas.clear_inpaint()

                    if(that.history_key ){
                        let p = {
                            "prompt": "Inpainting: " +  that.prompt , "key":that.history_key , "imgs" : [img_path] , "inp_img": that.input_img_orig,
                            "model_version": that.stable_diffusion.model_version , "guidence_scale" : that.guidence_scale , 
                        }                
                        Vue.set(that.app_state.app_data.history, that.history_key , p );
                    }
                    
                    

                    
                },
                on_progress(p){
                    that.done_percentage = p;
                    
                },
                on_err(err){
                    that.backend_error = err;
                },
            }

            this.is_stopping = false;


           if(this.stable_diffusion)
                this.stable_diffusion.text_to_img(params, callbacks, 'inpainting');
        } , 

        set_inp_image(img_path){
            this.inp_img = img_path;
            this.undo_history = []
            this.input_img_orig = img_path
            this.history_key = Math.random()+"inp"

            if(this.$refs.inp_img_canvas){
                this.$refs.inp_img_canvas.clear_inpaint()
            }
        },

        open_input_image(){
            if( !this.stable_diffusion.is_input_avail)
                return;
            let img_path = window.ipcRenderer.sendSync('file_dialog',  'img_file' );
            if(img_path && img_path != 'NULL'){
               this.set_inp_image(img_path)
            }
        },

        onDragFile(e) {
            if (!this.stable_diffusion.is_input_avail)
                return;
            if (!e.dataTransfer.files[0].type.startsWith('image/'))
                return;
            let img_path = e.dataTransfer.files[0].path;
            this.set_inp_image(img_path)
        },

        do_undo(){
            let a = this.undo_history.pop()
            if(a){
                this.inp_img = a;
                if(this.$refs.inp_img_canvas){
                    this.$refs.inp_img_canvas.clear_inpaint()
                }
            }
                
        },

        save_img(){
            let out_path = window.ipcRenderer.sendSync('save_dialog' );
            if(!out_path)
                return
            
            let org_path = window.ipcRenderer.sendSync('save_b64_image',  this.$refs.inp_img_canvas.get_img_b64()  );
            if(!org_path)
                return
            org_path = org_path.replaceAll("file://" , "")
            window.ipcRenderer.sendSync('save_file', org_path+"||" +out_path);
        },

        clear(){
            if(this.$refs.inp_img_canvas){
                this.$refs.inp_img_canvas.clear_inpaint()
            }
            this.inp_img = ""
            this.undo_history = []
            this.prompt = ""
            this.input_img_orig = ""
            this.history_key = ""
            this.retry_params  = undefined
        },

        stop_generation(){
            this.is_stopping = true;
            this.stable_diffusion.interupt();
        },
        
    },
}
</script>
<style>
 .inapint_container{

        border-color: rgb(206, 212, 218);
        border-width: 1px;
        border-style: solid;

        width:  80vw ; 
        height: calc(100vh - 230px) ;  
        margin-left:10vw ;
        margin-top: 30px; 

    }

    .bottom_toolbox{
        position: absolute;
        
        bottom:20px;
        width: 80vw ; 
        margin-left:10vw ;

        /* width: calc(100vw - 20px) ; 
        margin-left:10px ; */


    }

    .inpaint_textbox{
        float:left;
        max-width: calc(100% - 100px);
    }   

    @media (prefers-color-scheme: dark) {
        .inapint_container{
            border-color: #606060
        }
    }

</style>
<style scoped>
</style>