# Step 1

### What is Figma and why is it important to Locofy?
- Figma is a cloud-based design and prototyping tool used for creating user interfaces and collaborating with team members. It's light, powerful and provides full functionality to help designers complete their work.
- It is important to Locofy because it has a large number of users and provides the plugin mechanism, so it can help Locofy to reach users in need and work directly with the design.

### What are examples of other design tools that Locofy might want to integrate with?
- Adobe XD, Sketch, Axure RP

### Why would a design tool like Figma want to provide a feature like “Auto Layout: Wrap”?
- This function can help to make the responsive design. As I know, before that, designers spend a lot of time creating multiple screens just to explain how it works when applying resizing. Now with this function, it can help to reduce time and error (consistent design).
- I see a lot of comments on youtube talking about this function as the revolution or game-changer in design.

### What is the CSS equivalent of “Auto Layout: Wrap”?
```
display: flex;
flex-wrap: wrap;
```

# Step 2
- I clone the design of `https://www.skyscanner.net/` but ignore some elements to make it simpler.
- Link to the design: https://www.figma.com/file/TZi7i4m4gpjo9usorlLD6X/locofy?type=design&node-id=0%3A1&mode=design&t=PSh4disAVtwrSOrU-1
- It should look like this: 
<img src="https://github.com/thaiduongx26/locofy-design-challenge/blob/main/images/figma_wrap.png" width=75% height=75%>

# Step 3
- Because this is an open question so I will base on my observation about the Figma functions and the current pipeline of Locofy. I just have 1 day for get familiar with design on Figma and test very few cases using Locofy product so please correct me if I'm wrong.
## Problem definition
- I feel like even though I'm more familiar with that tool, setting the "auto layout: wrap" still takes time and it needs to be tested a lot. Currently, the feature "generate code" of Locofy seem to work fine if the design already completed all the setting to make the design responsive.
- So it'll be more useful if Locofy product can help to convert a normal design to a responsive design. It can help to reduce the effort and time for the designer. And after this feature, Locofy can think about recorrecting the bad responsive design.
- It should be applying for not only to Figma, but other design tools like Photoshop, Adobe XD, Sketch, ...

## Idea
- So we have the problem is converting a normal design to a responsive design. I have 2 ideas for that:
  - Option 1: Design2code(React, html, css):
    - Pros:
      - Minimal the steps.
      - Users feel it like magic.
      - Simpler AI pipeline.
      - Could take advantage of third-party models like gpt-4 when it's released.
    - Cons:
      - Require a huge data to train the LLM.
      - A lot of errors will appear and hard to reduce them.
      - Designer is designer, they care about the design first.
  - Option 2: Converting directly to the user's design:
    - Pros:
      - Users can actually see what changed in their design.
      - Every member can see and understand what steps need to be improved to improve the whole pipeline.
      - Easier to split the task.
    - Cons:
      - Multiple steps, hard to deploy and maintain.
      - One error in 1 step and affect to the whole pipeline.
     
- I think option 2 is more suitable and can be applied right away. So I'll focus on that in this proposal.

## Approach
### Input
- For Figma and some design tools, we can get the image, pages, artboards, layers, and their properties by using API. But each design tool has a different format, so we need to convert it to only one internal format. So we can run the AI pipeline for all the platforms.
- In conclusion, the input of AI should be image and property information. Example:
```
{
    "image": "link-to-image",
    "children": [
        {
            "id": "1:2",
            "properties": {
                "background": [...],
                "border": [...],
                ...
            },
            "children": [
                {
                    "id": "1:2:3",
                    "properties": {
                        "background": [...],
                        "border": [...],
                        ...
                    },
                    "children": [...]
                },
                ...
            ]
        },
        ...
    ]
}
```
### Output
- In Figma, when I convert a frame to "Auto layout: wrap", there are some differences that I can find:
  - In the parent frame:
    ```
    'layoutMode': 'HORIZONTAL',
    'itemSpacing': 95.0,
    'paddingLeft': 33.0,
    'paddingRight': 33.0,
    'paddingTop': 62.0,
    'paddingBottom': 62.0,
    'layoutWrap': 'WRAP',
    'counterAxisSpacing': 95.0,
    'counterAxisAlignContent': 'AUTO',
    ```
  - In the child frames:
    ```
    'layoutAlign': 'INHERIT',
    'layoutGrow': 0.0,
    ```
- That means if we can detect what frame needs to be applied `Auto layout: wrap`, we can apply the rule directly to make it responsive.
- In conclusion, we need a model that can predict the correct frames need to be responsive.
### Pipeline
<img src="https://github.com/thaiduongx26/locofy-design-challenge/blob/main/images/locofy-pipeline.png">

### Proposal models
#### Image-based
- We simply use a detection model like Yolo or Detectron to predict the layer that needs to convert to responsive.
- Because the output of the model is the bounding box of the object so we need a way to mapping the output to the correct layers. 
<img src="https://github.com/thaiduongx26/locofy-design-challenge/blob/main/images/image-based-model.png">

#### Layer-based
- Basically, a responsive layer always contains some child frame. So we can use the child's information to predict the class of the parent frame is responsive or not.
- Some information that we can use like: bounding box (x1,x2,y1,y2), image (crop based on bbox), and content (I assume that can be text).
- We use a Transformers encoder model and a classification head.
<img src="https://github.com/thaiduongx26/locofy-design-challenge/blob/main/images/layer-based-model.png" width=80% height=80%>

## Serving
### Deployment
- The deployment step should follow the pipeline below. We use `dvc` for control and versioning the model so it can ignore miss understanding between models appearing in the same space. We need to set up CI/CD to test the model in the dev environment to make sure no error of installation and it should pass all the test cases. After the new version is merged into the production branch, it needs to auto-deploy to the production.
<img src="https://github.com/thaiduongx26/locofy-design-challenge/blob/main/images/deployment-flow.png">

### Monitoring
- As my experiment, I recommend using `neptune.ai` for both model training, evaluation, logging, ... and `Grafana` for controlling the model performance in production.
- The new model should be served around 30% of users and investigate the user reaction by getting feedback, survey, rating, ... After the data tell confidence about new model, the old model will be replaced.
