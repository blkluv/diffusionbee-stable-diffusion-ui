@import '../assets/css/theme.css';
<template>
    <div  class="animatable_content_box ">
        <div v-if="stable_diffusion.is_backend_loaded">
            <div class="textbox_section" >
                <textarea 
                    v-model="prompt" 
                    placeholder="Enter your prompt here" 
                    style="border-radius: 12px 12px 12px 12px; width: calc(100%); resize: none; " 
                    class="form-control"  
                    v-bind:class="{ 'disabled' : !stable_diffusion.is_input_avail}"
                    :rows="is_negative_prompt_avail ? 2:3"></textarea>

                <textarea 
                    v-if="is_negative_prompt_avail"
                    v-model="negative_prompt" 
                    placeholder="Enter your negative prompt here" 
                    style="border-radius: 12px 12px 12px 12px; width: calc(100%); resize: none; margin-top: 5px; " 
                    class="form-control negative_prompt_tb"  
                    v-bind:class="{ 'disabled' : !stable_diffusion.is_input_avail}"
                    :rows="1"></textarea>

                <div v-if="stable_diffusion.is_input_avail" class="content_toolbox" style="margin-top:10px; margin-bottom:-10px;">
                    
                    <div class="l_button button_medium button_colored" style="float:right ; " @click="generate_from_prompt">Generate</div>

                    <!-- <div style="float:right;"  >
                        <div class="l_button" @click="is_adv_options = !is_adv_options">Advanced options</div>
                    </div> -->

                    <SDOptionsDropdown :options_model_values="this_object"  :elements_hidden="[ 'inp_img_strength' ]"> </SDOptionsDropdown>


                    <div style="float:right; margin-top: -5px; " >
                        <b-dropdown id="dropdown-form" variant="link" ref="dropdown" toggle-class="text-decoration-none" no-caret >
                        
                            <template #button-content>
                                <div class="l_button"  style="margin-right: -20px;" >Styles</div>
                            </template>

                            <b-dropdown-form style="width: 540px ; ">

                                <div style="max-height: calc(100vh - 300px); overflow-y: scroll;">
                                    <div v-for="modifier in modifiers" :key="modifier[0]">
                                        <p>{{modifier[0]}}</p>
                                        <div v-for="tag in modifier[1]" @click="add_style(tag)" :key="tag" class="l_button button_small button_colored" style="float:left; margin-bottom: 7px;">{{tag}}</div>

                                        <div style="clear: both; display: table; margin-bottom: 3px;"> </div>
                                        <hr>
                                    </div>

                                    <p>Source : cmdr2</p>
                                </div>

                                
  
                            </b-dropdown-form>
                        </b-dropdown>
                    </div>

                    <div class="l_button button_medium" style="float:right ;margin-right: -10px; margin-top: -1px;" @click="open_arthub">Prompt Ideas</div>


                    
                </div>
                <div v-else-if="stable_diffusion.generated_by=='txt2img'"  class="content_toolbox" style="margin-top:10px; margin-bottom:-10px;">
                    <div v-if="is_stopping" class="l_button button_medium button_colored" style="float:right" @click="stop_generation">Stopping ...</div>
                    <div v-else class="l_button button_medium button_colored" style="float:right" @click="stop_generation">Stop</div>
                </div>
            </div>


            <div v-if="generated_images.length == 1" >
                <center>
                    <ImageItem :app_state="app_state"  :path="generated_images[0]" :style_obj="{ 'width': 'calc(100vh - 390px )' , 'margin-top': '60px' }"></ImageItem>
                </center>
            </div>
            
            <div>
                <br> 
                <b-row v-if="generated_images.length > 1"  class="justify-content-md-center" >

                    <b-col  v-for="img in generated_images" :key="img" style="margin-top:80px"  md="6" lg="4" xl="3"  >
                        <center>
                            
                            <ImageItem :app_state="app_state"  :path="img" :style_obj="{'max-width' :'85%'}"></ImageItem>
                        </center>
                    </b-col>
                        

                
                </b-row>
                <br> 
            </div>

            <div v-if="backend_error" style="color:red ; margin-top:50px;">
                <div class="center loader_box">
                     <p>Error: {{backend_error}} </p>
                </div>
            </div>
        </div>

        <div v-if="!stable_diffusion.is_input_avail && stable_diffusion.generated_by=='txt2img'">
            <LoaderModal :loading_percentage="done_percentage" loading_title="Generating" :loading_desc="stable_diffusion.generation_state_msg" :remaining_times="stable_diffusion.remaining_times"></LoaderModal>
        </div>
    
    <div class="bottom_float">
        <p>Please close other applications for best speed.</p>
    </div>

    <b-dropdown v-if="generated_images.length > 0" left variant="link" size="sm" toggle-class="text-decoration-none" no-caret style="float:right; margin-top: 5px;">
        <template #button-content>
            <div class="l_button bottom_float" style="right : 10px; bottom : 15px; background-color: inherit; cursor: pointer;">
                Share 
            </div>
        </template>
        <b-dropdown-item-button @click="share_current_arthub">Share on ArtHub.ai</b-dropdown-item-button>
        <b-dropdown-item-button @click="share_current_w3up">Share on web3.storage</b-dropdown-item-button>
    </b-dropdown>

</div>



</template>
<script>

import LoaderModal from '../components_bare/LoaderModal.vue'
import Vue from 'vue'
import ImageItem from '../components/ImageItem.vue'
import {share_on_arthub, share_on_w3up} from '../utils.js'
import SDOptionsDropdown from '../components_bare/SDOptionsDropdown.vue'

export default {
    name: 'ImgGenerate',
    props: {
        app_state : Object   , 
        stable_diffusion : Object,
    },
    components: { LoaderModal, ImageItem, SDOptionsDropdown },
    mounted() {
       
    },
    data() {
        return {
            img_w : 512, 
            img_h : 512 , 
            dif_steps : 25,
            guidence_scale : 7.5 , 
            is_adv_options : false , 
            seed : ""  , 
            prompt : "",
            num_imgs : 1,
            batch_size : 1 , 
            generated_images : [],
            backend_error : "",
            done_percentage : -1,
            is_stopping : false,
            modifiers : require("../modifiers.json"),
            is_negative_prompt_avail : false, 
            negative_prompt : "",
            selected_model : 'Default'
        };
        
    },
    computed: {
        this_object(){
            return this;
        }
    },
    methods: {
        generate_from_prompt(){
            let seed = 0;
            if(this.seed)
                seed = Number(this.seed);
            else
                seed = Math.floor(Math.random() * 100000);

            this.computed_seed = seed;


            let params = {
                prompt : this.prompt , 
                W : Number(this.img_w) , 
                H : Number(this.img_h) , 
                seed :seed,
                scale : this.guidence_scale , 
                ddim_steps : this.dif_steps, 
                num_imgs : this.num_imgs , 
                batch_size : this.batch_size 
            }

            if(this.selected_model && this.selected_model != "Default" && this.app_state.app_data.custom_models[this.selected_model] ){
                params.model_id = -1;
                params.custom_model_path =  this.app_state.app_data.custom_models[this.selected_model].path;
            }

            if(this.is_negative_prompt_avail)
                params['negative_prompt'] = this.negative_prompt;

            let that = this;

            if(this.prompt.trim() == ""){
                Vue.$toast.default('You need to enter a prompt to generate an image')
                return;
            }
                

            this.backend_error = "";
            Vue.set(this,'generated_images' ,[]);
            this.done_percentage = -1;

            let history_key = Math.random();

            let callbacks = {
                on_img(img_path){
                    that.generated_images.push(img_path);

                    if(!(that.app_state.app_data.history[history_key])){
                        let p = {
                            "prompt":that.prompt , "seed": seed, "img_w":that.img_w , "img_h":that.img_h ,  "key":history_key , "imgs" : [],
                            "guidence_scale" : that.guidence_scale , "dif_steps" : that.dif_steps 
                        }
                        if(that.stable_diffusion.model_version)
                            p['model_version'] = that.stable_diffusion.model_version;
                        if(that.is_negative_prompt_avail)
                            p['negative_prompt'] = that.negative_prompt;
                        Vue.set(that.app_state.app_data.history, history_key , p);
                    }
                        

                    
                    that.app_state.app_data.history[history_key].imgs.push(img_path)

                    console.log(that.app_state.app_data.history)

                },
                on_progress(p ){
                    that.done_percentage = p;                
                },
                on_err(err){
                    that.backend_error = err;
                },
            }

            this.is_stopping = false;


           if(this.stable_diffusion)
                this.stable_diffusion.text_to_img(params, callbacks, 'txt2img');
        } , 


        open_arthub(){
            window.ipcRenderer.sendSync('open_url', "https://arthub.ai");
        },

        stop_generation(){
            this.is_stopping = true;
            this.stable_diffusion.interupt();
        },

        add_style(tag){
            this.prompt += ", " + tag;
        },

        share_current_arthub(){
            this.app_state.global_loader_modal_msg = "Uploading";
            let params =  {
                "Img Width": Number(this.img_w) , 
                "Img Height" : Number(this.img_h) , 
                Seed :this.computed_seed,
                Scale : this.guidence_scale , 
                Steps : this.dif_steps ,
                model_version: this.stable_diffusion.model_version
            }
            if(this.is_negative_prompt_avail)
                params['Negative Prompt'] = this.negative_prompt;

            let that = this;
            share_on_arthub( that.generated_images , params , that.prompt).then((
                function(){ that.app_state.global_loader_modal_msg = ""}
            )).catch(
                function(){alert("Error in uploading.") ; that.app_state.global_loader_modal_msg = ""}
            )
        },
        
        share_current_w3up(){
            let params =  {
                "Img Width": Number(this.img_w),
                "Img Height" : Number(this.img_h),
                Seed :this.computed_seed,
                Scale : this.guidence_scale,
                Steps : this.dif_steps,
                model_version: this.stable_diffusion.model_version,
            }
            
            share_on_w3up( this.generated_images, params, this.prompt)
        }

    },
}
</script>
<style>
   .center {
      margin: 0;
      position: absolute;
      top: 50%;
      left: 50%;
      -ms-transform: translate(-50%, -50%);
      transform: translate(-50%, -50%);
    }

    .disabled{
        pointer-events:none;
        opacity: 0.5;
    }

    .loader_box{
       padding: 20px;
        /*height: calc(160px);*/
        border-radius: 12px 12px 12px 12px;
    }


    .ad_form_box{
        float:left; 
        background-color: rgba(0, 0, 0, 0.1); 
        padding:10px ; 
        margin-right: 10px;
         border-radius: 3px 3px 3px 3px;
    }

    .gal_img{
        border-radius: 12px 12px 12px 12px;
        /* background-color: white; */
        border-style: solid;
        border-width: 1px;
        border-color: rgba(0, 0, 0, 0.1);
        cursor: pointer;
        box-shadow: 0px 0px 1.76351px rgba(40, 41, 61, 0.04), 0px 3.52703px 7.05405px rgba(96, 97, 112, 0.16);

    }

    @media (prefers-color-scheme: dark) {
        .gal_img{
            border-color: rgba(255, 255, 255, 0.3);
        }
    }

    .negative_prompt_tb{
        color:brown !important;
    }

    @media (prefers-color-scheme: dark) {
        .negative_prompt_tb{
            color:rgb(210, 0, 0) !important ;
        }

    }

</style>
