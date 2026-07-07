# PlateCalc---Food-Calorie-Estimator

PlateCalc estimates the total calorie content of a meal from a single photo. It uses a fine-tuned Ultralytics YOLOv8 instance segmentation model to detect and segment individual food items on a plate, estimates the weight of each item, and looks up its calorie density from the USDA FoodData Central (FDC) API to compute an estimated calorie breakdown.

How It Works
Dataset preparation :
The FoodSeg103 dataset (raw image + pixel-mask annotations) is converted from its native mask format into YOLO segmentation format (normalized polygon coordinates per class), split into train/val sets.
Model training :
A yolov8n-seg model is trained (or fine-tuned from a checkpoint) on the converted dataset across 103 food categories (fruits, vegetables, meats, grains, desserts, beverages, etc.), producing a best.pt weights file.
Inference & segmentation :
The trained model runs instance segmentation on a new plate photo (with EXIF-orientation and color-mode correction applied first), returning a mask and class label for each detected food item.

Weight estimation
Each detected class is handled one of two ways:
Countable items (e.g. banana, egg, apple): connected-component analysis on the mask counts discrete instances, multiplied by an average weight per item.
Area-based items (e.g. rice, soup, sauce): the pixel area of the mask is converted to grams using a per-food density factor.

Calorie lookup — For each food class, calories per 100g are fetched from the USDA FDC API (first by a known fdc_id, falling back to a name search, and finally to a generic 140 kcal/100g estimate if no match is found). Results are cached to avoid repeat API calls.
Output — A pandas DataFrame lists each detected item, its estimated quantity/weight, kcal per 100g, and total calories, sorted by calorie contribution. A side-by-side plot shows the segmented detections and a pie chart of calorie contribution per item.

Notes & Limitations
The model could not be trained over higher epochs, so there is a considerable loss associated with it ( I ran out of colab T4 GPU limits twice ).
Weight estimates are heuristic (average item weights and per-pixel density factors calibrated for typical overhead plate photos), not precise measurements actual calorie counts will vary with portion size, camera angle, and plating.
Foods without a known FDC ID or USDA match fall back to a generic 140 kcal/100g estimate.
Each user must supply their own USDA API key (free tier).
