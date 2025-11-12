# Vision Language Time

<img width="4800" height="1200" alt="vlt_header" src="https://github.com/user-attachments/assets/3b5e9b49-3899-4b2c-83c4-e51a49181c07" />

Inspiration for this project idea came about from discussions around LLMs as “decision engines” and recent research papers that were using LLM-as-judge for their validations that I reviewed as part of NeurIPS 2025.

*If we trust LLMs for everything else, why not for interpreting time itself?*  

---

## Overview
I created a conceptual and working prototype of a new clock: **VLT (Vision-Language-Time)**, where time itself is interpreted by a vision-language model running on the Raspberry Pi.

## Proposal

For Part 2, I propose **Vision-Language-Time (VLT)**:  

Instead of a standard digital or analog clock, the Raspberry Pi 5 runs a small VLM (ex. Moondream/FastVLM scale) locally. As often as it can (due to compute limitations), it captures an image from its camera and asks the VLM: *“What time is it?”*. The model outputs its “perceived time,” which is displayed on the PiTFT screen along with the ground-truth system time.  

We also log:  
- The image frame  
- The VLM’s predicted time  
- The true time  

This gives us the ability to perform an analysis of accuracy afterwards. We can tag images as “indoors” vs. “outdoors” (or other contextual tags) to see if environment affects performance (like artificial vs. natural light), etc.

The questions we can explore:  
- How accurate is the VLM at telling time?  
- Are we ready to replace traditional timekeepers with AI perception?  
- Could trust in such a clock be measured in user studies?  

**Sketch**  
<p align="center">
<img height="300" src="https://github.com/user-attachments/assets/a8aa2600-d298-4a1c-8271-d537bce888ee">
<img height="300" alt="verplan_vltpng" src="https://github.com/user-attachments/assets/4972c1b2-2ee5-419d-bac7-d4650ac73a9b" />
<p align="center">
  
One of the above sketches was made with AI as a double-meaning joke for "We trust AI for everything...", can you tell?

## Components

I created `screen_clock_vlm.py` based on `screen_clock.py` to for our **VLT pipeline**. Instead of just printing system time, the script captures an image via connected webcam, passes it to the local VLM, and shows both the **predicted “AI time”** and the **real time** side by side.  

You might ask some questions like:
**Does time have to be linear?  How do you measure a year? [In daylights? In midnights? In cups of coffee?](https://www.youtube.com/watch?v=wsj15wPpjLY)**

Time is measured based on the compute speed of our VLM/LLM (and compute restrictions of the RPI)- so about once every ~10-20 seconds. We measure the time with the current image's lighting conditions. Obviously this is fed into the VLM/LLM so we are only getting the "scraps" or downstream of whatever the quality/training of the models we use are. However, that is the point of the project in itself, yielding control over something relatively simple (like calculating time) to a trained program.

> #### Notice:
> I had to modify the idea slightly since VLMs tend to be trained on lots of images of clocks with "10:10" or "12:12". This led to prompts that ask the VLM for time nearly always resulting in these times. So, to fix this, I instead ask the VLM for what it is good at: a description of the image, specifically the > lighting conditions. Then, I pass the VLM's description of the image to a local Qwen 0.5B Instruct model running via Ollama on the RPI5 and it guesses the time for us.

**Some examples of VLM failure (asking directly for time):**

<div align="center">
  <table>
    <tr>
      <td align="center">
        <img src="https://github.com/user-attachments/assets/c3caf0ba-c712-407f-b692-6c8125ba0d07" width="300"/><br>
        <b>VLM Time:</b> 10:10<br>
        <b>Actual Time:</b> 16:11
      </td>
      <td align="center">
        <img src="https://github.com/user-attachments/assets/6d38ada6-72fa-4ad6-9881-12e08d596064" width="300"/><br>
        <b>VLM Time:</b> 10:10<br>
        <b>Actual Time:</b> 16:11
      </td>
    </tr>
  </table>
</div>

The exact prompt used on the VLM is:
```
Describe only observable lighting cues. Describe environment/sky/weather; natural light (direct vs diffuse, where it enters, sun patches/glare); shadows (presence, edge sharpness, relative length, direction); artificial lights (which sources are on, brightness low/medium/high, color warm/neutral/cool); overall brightness/exposure (very dark/dim/medium/bright, blown highlights, deep shadows, noise, motion blur); windows/openings and orientation hints; secondary clues (streetlights on, blinds/shades state, screen glow); brief caveats/confidence.
```

The exact prompt used on the Instruct model is:
```
You are a time estimator. Based ONLY on the following visual/lighting description, "
        "estimate the local clock time as HH:MM in 24-hour format. "
        "If uncertain, give your BEST plausible estimate. "
        "Output ONLY the time in the format HH:MM. No words, no seconds, no explanations.\n\n"
        "Description:\n"
        f"{vlm_text.strip()}\n\n"
        "Answer:\n"
```

## Results

I created the following scripts for this project:
- **screen_clock_vlm.py**: inference for program, will run our app that takes an image, captions it, passes it to local LLM, and then shows the predicted time and real time (as well as saving the image, times, etc. in bg)
- **fastvlm_server.mjs**: local FastVLM server for RPI5

To run, I recommend a venv for installing the reqs to inference the HF model. All you need to do is run `python screen_clock_vlm.py`. You can add an optional -o argument if you are running the VLM somewhere else (it is very slow on RPI5, about ~15s per request, I ended up using a CF tunnel and my PC to get this much faster- see code for where you can input your tunnel URL).

Please see videos below for a demonstration.

**Video of VLT pipeline:**

https://github.com/user-attachments/assets/7b9f53a8-2b60-4965-a5ec-6f75ebe499de

**Video of VLT interaction with last image, last VLM output, time screen:**

https://github.com/user-attachments/assets/8d9d3851-ec70-4a5a-9037-bad438787a62

**Some examples of usage:**


| Image | VLM Description | Perceived Time | Actual Time |
|-------|-----------------|----------------|-------------|
| ![Image 2](https://github.com/user-attachments/assets/89d06a83-db95-43e2-ac32-0c44428fd44f) | does not provide clear evidence of natural light sources like windows or outdoor elements, but the lack of brightness suggests it might be either an overexposed photograph or a room with minimal natural light. The overall impression is one of tranquility and stillness. As for the second part of your question, there are no discernible natural light indicators such as sunbeams, shadows, or glare that would confirm the presence of direct or indirect natural light in the room at the time the photo is taken. | 01:00 | 16:52 |
| ![Image 1](https://github.com/user-attachments/assets/40698baa-ed94-41ae-a73e-978cbfa695f6) | lighting being on, as there are no visible lights turned on. The windows are open, as evidenced by the visible curtain and the way the light is entering the room, but the curtains are drawn back, allowing for unobstructed light to enter. The sky outside is clear, with no visible clouds, suggesting fair weather conditions. | 15:00 | 16:52 |
| ![Image 3](https://github.com/user-attachments/assets/5a359655-fe5a-4cc0-aaa7-9e64967a99e8) | high-rise buildings and smaller structures, possibly a downtown area. The orientation of the buildings and the angle of the shot suggest that this is a view from a high vantage point, such as a skyscraper or a tall building. The lack of any visible movement or activity in the scene implies a moment of stillness, perhaps early morning or late afternoon. | 06:00 | 16:54 |
| ![Image 1](https://github.com/user-attachments/assets/d0dc26be-2e6a-40f1-8cb1-56d3a2723be8) | sun provides ample light. The image does not show any motion blur, indicating that the camera was still when the photo is being taken. The windows' state is not entirely clear due to the angle and focus of the shot, but they do not appear to be open, as there is no visible gap between the window and the frame. | 09:30 | 16:55 |
| ![Image 4](https://github.com/user-attachments/assets/114ef6d3-80ee-4ac7-a68f-8fe288b86424) | with ambient indoor lighting. The lack of sharp shadows and the presence of soft edges throughout the image contribute to a calm and serene atmosphere. There is no indication of motion blur or other photographic effects, and the image does not provide any clues about the time of day beyond the general impression of daytime. The simplicity of the composition focuses attention on the texture of the material, rather than any specific environmental details. | 09:00 | 16:58 |
| ![Image 1](https://github.com/user-attachments/assets/27cae21b-c109-4bad-9702-8aebb17685e6) | Not appear to be outdoors, given the lack of natural elements like trees or sky. The decor, including the posters, is consistent with a personal space, possibly a living room or a bedroom, where one might relax and enjoy the ambiance created by the lighting. The presence of the lamp and the style of the posters suggest a preference for a certain aesthetic or thematic decor, which could be reflective of the occupant's personal taste or interests. | 08:30 | 17:06 |

## Discussion

The Vision Language Time project shows what happens when we let an AI interpret something as ordinary as time through what it sees instead of what it measures. The results remind us that AI systems don’t “know”; they guess, based on patterns they’ve learned. The repeated “10:10” predictions aren’t random errors; they reveal how training data shapes what the model expects to see. In a sense, the model isn’t telling time, it’s describing a world it has learned from other images of clocks. This makes VLT a small but meaningful experiment in understanding how machines perceive and misinterpret everyday concepts.

From a technical standpoint, the project highlights both the promise and the limits of running small vision-language models locally. These models can describe lighting, color, and environment well, but they struggle to connect those visual clues to specific times of day. Passing the VLM’s description to a smaller language model improves the guess slightly, showing how visual and language reasoning can complement each other. Even so, the pipeline runs slowly and is far from accurate, but that’s part of what makes it interesting. It shows how perception, reasoning, and computation interact under real-world constraints.

In the end, this project is as much about trust as it is about time. Watching a clock that confidently gives the wrong answer is a reminder of how easily we project confidence and intelligence onto machines. VLT doesn’t fail, it simply perceives the world in its own limited way. By turning over something as simple as timekeeping to an AI, we get a glimpse of what it means to rely on systems that interpret, rather than truly understand, the world around them.

