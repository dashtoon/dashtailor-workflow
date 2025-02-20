# dashtailor-workflow
<a href="https://github.com/dashtoon/dashtailor-workflow/blob/main/Dashtoon%20clothes%20and%20object%20transfer%20v4.json">ComfyUI Workflow</a> for Training Free Clothing and Object Transfer in Images

This workflow enables you to reliably transfer any object from an image into a target image while maintaining the scene context of the target image. This is particularly useful for transferring clothing onto a character. 
![clothes transfer example for gradio](https://github.com/user-attachments/assets/4af61149-08bb-4e17-9811-b7aa3876dac6)
![final inpainted grid](https://github.com/user-attachments/assets/ee8c8a12-98e9-4456-a172-b7e215abb13e)

## **Models**
<a href = "https://huggingface.co/black-forest-labs/FLUX.1-Fill-dev">Flux Fill dev</a> 

Optionally, <a href = "https://huggingface.co/black-forest-labs/FLUX.1-Redux-dev">Flux Redux dev</a>

<a href = "https://huggingface.co/Shakker-Labs/FLUX.1-dev-ControlNet-Union-Pro">Flux Controlnet Union Pro</a>

This workflow makes a call to OpenAI GPT-4o API using the NegiTools custom node. You will need to add OPENAI_API_KEY in the .env file located in ComfyUI's root directory. Make sure you are hitting the model gpt-4o-2024-11-20 or later for best results. Add it in the NegiTools OpenAI GPT node's openai_gpt.py file if it is not there already.
<p align="center">
   <img src="https://github.com/user-attachments/assets/4b09aaf4-3994-40c9-abca-be3a38c75b53" width="50%">
  </p>

While making comics or any other form of visual storytelling using generative AI, achieving consistency in clothing and specific objects across panels is essential. It’s easier to do with characters and art styles by using LoRAs. But it’s impractical to train a LoRA for clothing because using too many concept LoRAs together creates unwanted artifacts. A character may might wear different outfits in different scenes, so generating the clothing as a part of the character by including it consistently in every image of the LoRA dataset isn’t practical either. We need a method to upload an image of clothing or an object and transfer it seamlessly into the target image.

To solve this problem we used an interesting capability of the Flux Fill inpainting model that transfers concepts from one part of an image to another part of the *same image* remarkably well.

This repository contains a ComfyUI workflow for achieving this. The explanation of the workflow is below. The workflow does not include the final inpainting pass to improve quality because that needs the checkpoint and generation configuration specific to the target image’s style. Feel free to add that on your own. 
![clothes transfer workflow image](https://github.com/user-attachments/assets/daf0ff42-3f4d-4666-bebe-8d3e2273c263)


## **Approach**

1. **Masking:** Draw masks on the
    - Object image – the mask covers the piece of clothing the character needs to wear
      | Object image | Object mask |
      |---------|---------|
      | ![object iamge](https://github.com/user-attachments/assets/548788ae-e64a-4804-9411-2a97d12e1e77) | ![object mask](https://github.com/user-attachments/assets/a265ef6a-02dc-4354-a440-8313b2c6c833) |
    
    - Target image – the mask covers should cover only the part of the image where the object must be applied, such as the character’s body
    
    | Target image | Target mask |
      |---------|---------|
      | ![target image](https://github.com/user-attachments/assets/229d46b4-3711-4b08-b30d-cda340e182cf) |![target mask](https://github.com/user-attachments/assets/13e64550-509b-4b88-9c4a-f3117df6faf8) |
    
2. **Extracting the Masked Object:** Isolate the part of the object image within the mask, leaving the rest of the image blank. The original aspect ratio of the image was retained.
  <p align="center">
   <img src="https://github.com/user-attachments/assets/89c6c37e-fa3d-443f-944d-79354e9710b4" width="50%">
  </p>
  
3. **Using GPT-4o Vision:** Describe the isolated object to use as a prompt. The GPT instruction prompt is: `Describe the clothing, object, design or any other item in this image. Be brief and to the point. Avoid starting phrases like "This image contains...”`
In this case the extracted object prompt was `Metallic plate armor with intricate designs, including a winged emblem on the chest. Brown leather straps and accents secure the armor, complemented by layered pauldrons and arm guards.` 
4. **Creating a Composite Image:** Join the object image and the target image side by side. Scale the object image height up or down to match the target image.
   <p align="center">
   <img src="https://github.com/user-attachments/assets/b0163e03-219d-4116-862a-0130d4ef90a4" width="50%">
  </p>

5. Pose controlnet: To maintain the pose of the original subject while inpainting, we passed the composite image into an Openpose annotator and used Flux Union Controlnet with type Openpose and strength value of 0.8.
    <p align="center">
   <img src="https://github.com/user-attachments/assets/013460c0-3b7a-435a-ae95-e05ed1669fe3" width="50%">
  </p>

6. **Flux Fill Inpainting:** Use the composite image along with the GPT extracted prompt as the conditioning to guide the inpainting process. The inpainting parameters are:
    1. Denoise: 1
    2. Steps: 20
    3. CFG: 1
    4. Sampler: Euler
    5. Scheduler: Beta
    
    This seamlessly transfers the object into the masked area.

   <p align="center">
   <img src="https://github.com/user-attachments/assets/10970235-17f3-450d-9477-a53c08d9def2" width="50%">
  </p>
    
7. Cropping the composite image: Crop the output from the left by the width of the scaled object mask image to get the target image with the transferred output.
    <p align="center">
   <img src="https://github.com/user-attachments/assets/41b6c465-6b52-4f7b-9628-0d5157985a20" width="50%">
    </p>
    
8. This output may still have some rough edges and artifacts. Also there may be a style mismatch if the object image was in a different art style than the target image. So we recommend doing one final inpainting pass at 0.15 to 0.2 denoise and the same object prompt from earlier. It’s important that this inpainting pass uses the checkpoint, Lora, and generation configuration that’s suitable for the target image style. This will eliminate any artifacts and ensure style consistency. For example, if your target image is in anime style, use an anime checkpoint or Lora for this step.
    <p align="center">
     <img src="https://github.com/user-attachments/assets/c1e22d4d-cd1f-42f0-b22a-eefa131f84d3" width="50%">
    </p>

9. Optionally, you can use the Flux Redux model instead of relying on GPT to write a prompt based on the masked object. To use Redux, change the value in this node in the _Generation parameters_ section  from 1 to 2. However, Redux is tricky to use. If your object and target masks do not perfectly match, black patches may be generated. For a general use case I would recommend using a prompt.
   <p align="center">
     <img src="https://github.com/user-attachments/assets/b6d911b7-2c57-4722-97e4-680d57ea78ad" width="50%">
    </p>

## **Why Does This Work?**

When you combine the object and target images into a single composite image, you’re providing the model with a unified context. This allows the model to leverage the spatial and visual cues from both images simultaneously. Flux Fill is excellent at inpainting because its architecture utilizes the context of the image very well. 

When both the object and the target areas are present in the same composite image, the model can more effectively learn the relationship and ensure the masked area is filled in a way that aligns with the object’s characteristics and the overall scene. The model’s attention mechanism can more easily correlate the visual features of the object with the masked area. This ensures a more precise and accurate transfer, effectively guiding the model to fill in the missing parts with high fidelity.

This method effectively transfers the isolated object into the target image, maintaining consistency across different poses and orientations. 

## How to use DashTailor

You can run this workflow in CommfyUI, as explained above. 
DashTailor is also available to use on dashtoon.com/studio. Just Create a new Dashtoon and use the tool in the Editor section.

## More examples
![2](https://github.com/user-attachments/assets/013b5f91-6c3e-48ff-82e8-3364d73c92ca)
![1](https://github.com/user-attachments/assets/4bd6ebf3-8794-47e9-8d09-247e58abf3f1)
![3](https://github.com/user-attachments/assets/f4f166cf-cf80-4510-bcd7-95675a285f49)
![4](https://github.com/user-attachments/assets/bbb38d93-f2e7-4f64-8213-788af846437b)
![5](https://github.com/user-attachments/assets/d7c6ed5f-a895-44d7-bfca-67b68f2f5162)

