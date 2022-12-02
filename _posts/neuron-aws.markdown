layout: post
title: "Neuron instruction"
date: 2023-12-02 03:03:03 -0000
categories: 

### First steps:

1. Step-up machine for working on aws neuron

​		https://awsdocs-neuron.readthedocs-hosted.com/en/latest/neuron-intro/pytorch-setup/pytorch-install.html#install-neuron-pytorch

2. Activate environment

   `source activate aws_neuron_pytorch_p36`

   

### How to convert models to aws neuron

1. import libraries need to load model + model itself

2. Set model to eval

   `model.eval()`

3. Create example to feed into model - with the right dimensions, eg:

   `image = torch.zeros([1, 3, h, w], dtype=torch.float32)`

4. Trace model in aws neuron

   `model_neuron = torch.neuron.trace(model, example_inputs=[image])`

5. Save model

   `model_neuron.save("person-original-neuron.pt")`

6. Compare results to that of the original model:

   `#Run inference with original model`

   `tt1 = time.time()`
   `output_cpu = model(image)`
   `print('Inference time for cpu model is: ' + str(time.time()-tt1))`

   

   `#Run inference with neuron model`

   `tt2 = time.time()`
   `output_neuron = model_neuron(image)`

   `print('Inference time for neuron model is: ' + str(time.time()-tt2))`

   

   `#Verify that the CPU and Neuron predictions are the same by comparing the top-5 results`

   `top5_cpu = output_cpu[0].sort()[1][-5:]`
   `top5_neuron = output_neuron[0].sort()[1][-5:]`

   

   `#Lookup and print the top-5 labels`

   `top5_labels_cpu = [idx2label[idx] for idx in top5_cpu]`
   `top5_labels_neuron = [idx2label[idx] for idx in top5_neuron]`
   `print("CPU top-5 labels: {}".format(top5_labels_cpu))`
   `print("Neuron top-5 labels: {}".format(top5_labels_neuron))`



#### Specific models

For all the models that need some level of modification the modifications applied can be found in the [folder](/home/sara/Documents/model_optimization/my_tests) under the specific name of the model: 

- Kapao -> can't be fully traced as for now and some modification to the code is needed.

- Deep_sort -> can be fully traced without code modification. However, modification is needed in the file *deep_sort/deep/feature_extractor.py* for deployment. Since we are forcing a specific batch size, we will always be forced to infer that batch size. Therefore, if less people than the batch size are detected we will need to complete with zero tensors and then get rid of them after inference. If more people than the batch size we will need to split them into several calls.

- Yolov5 -> can be fully traced and the modifications need are included in the folder.
  - Person model weights a lot maybe if we substitute it for our own trained model it will be faster. Also, training kapao from our modified model might be faster and end up weighting less (?)
  
- 4xmpsn50 -> can be fully traced but several modifications need to be made:

  - In */home/ubuntu/mmpose/mmpose/models/detectors/top_down.py* -> we need to return the output_heatmap and comment the .decode() part (this will be done afterwards and won't be compiled in neuron)

      def forward_test(self, img, img_metas, return_heatmap=False, **kwargs):
          """Defines the computation performed at every call when testing."""
          batch_size, _, img_height, img_width = img.shape
          result = {}
          
          features = self.backbone(img)
          if self.with_keypoint:
              output_heatmap = self.keypoint_head.inference_model(
                  features, flip_pairs=None)
              print(output_heatmap.shape)
              return output_heatmap
          
          #if self.with_keypoint:
          #    keypoint_result = self.keypoint_head.decode(
          #        img_metas, output_heatmap, img_size=[img_width, img_height])
              #result.update(keypoint_result)
          
              if not return_heatmap:
                  output_heatmap = None
          
              #result['output_heatmap'] = output_heatmap
              #result = torch.Tensor(result['preds'])
          
          return output_heatmap#result

  - In *mmpose/models/backbones/mspn.py*, *class UpsampleUnit(nn.Module)* -> set align_cornes = False (maybe this is not necessary and only the latter is)
  - In *mmpose/models/heads/topdown_heatmap_multi_stage_head.py*,  *class PredictHeatmap(nn.Module)* set align_corners = False

      def forward(self, feature):
          feature = self.conv_layers(feature)
          output = nn.functional.interpolate(
              feature, size=self.out_shape, mode='bilinear', align_corners=False)
          if self.use_prm:
              output = self.prm(output)
          return output

- Vipnasresnet50 -> dimensions fail

- ResNetV1d-101 ---> INFO:Neuron:Exception = <class 'bool'>

- TSSTG (action recognitor)

  - Several issues regarding the required_grad = True in the parameters (weights and biases) of the layers of the model -> The solution is to get set required_grad = False for each of the types of parameters for each the layers. The file in which all these changes need to happen is Actionrecognition/Models.py and can be found in the model folder. The modified file will be saved in the save folder under Models_TTSG_neuron.py

  - Some of the first operations happening before inference is given make the tensor static so we need to get them outside of the model and have them as a preprocessing ActionEstLoader.py my change to this:

        def predict(self, mot, pts):
            """Predict actions from single person skeleton points and score in time sequence.
            Args:
                pts: (numpy array) points and score in shape `(t, v, c)` where
                    t : inputs sequence (time steps).,
                    v : number of graph node (body parts).,
                    c : channel (x, y, score).,
                image_size: (tuple of int) width, height of image frame.
            Returns:
                (numpy array) Probability of each class actions.
            """
            mot = mot.to(self.device)
            pts = pts.to(self.device)
        
            out = self.model((pts, mot))
        
            return out

     And the following operation will be preprocessing steps:

        pts[:, :, :2] = normalize_points_with_size(pts[:, :, :2], image_size[0], image_size[1])
        pts[:, :, :2] = scale_pose(pts[:, :, :2])
        pts = np.concatenate((pts, np.expand_dims((pts[:, 1, :] + pts[:, 2, :]) / 2, 1)), axis=1)
        
        pts = torch.tensor(pts, dtype=torch.float32)
        pts = pts.permute(2, 0, 1)[None, :]
        
        mot = pts[:, :2, 1:, :] - pts[:, :2, :-1, :]

  - The model also needs to be wrapped up and all of this can be found under */home/sara/Documents/model_optimization/neuron/convert_action_model.py*. Before running this file all of the previous changes above need to be in place -> replace Models.py by Models_TSSTG_neuron.py and update the ActionEstLoder.py with the code above.

        class NeuronWrapper(torch.nn.Module):
            def __init__(self, model, frame_shape):
                super().__init__()
                self.model= model
        def forward(self, pts):
            return self.model.predict(pts[:, :2, 1:, :] - pts[:, :2, :-1, :], pts)
