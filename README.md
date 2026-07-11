PlateCalc estimates the total calorie content of a meal from a single photo. It uses a fine-tuned Ultralytics YOLOv8 instance segmentation model to detect and segment individual food items on a plate, estimates the weight of each item, and looks up its calorie density from the USDA FoodData Central (FDC) API to compute an estimated calorie breakdown.

How It Works :

Dataset preparation: The FoodSeg103 dataset (raw image and pixel-mask annotations) is converted from its native mask format into YOLO segmentation format (normalized polygon coordinates per class), and split into train/val sets.

Model training: A yolov8n-seg model is trained (or fine-tuned from a checkpoint) on the converted dataset across 103 food categories (fruits, vegetables, meats, grains, desserts, beverages, and so on), producing a best.pt weights file.

Inference and segmentation: The trained model runs instance segmentation on a new plate photo (with EXIF orientation and color-mode correction applied first), returning a mask and class label for each detected food item.

Scale calibration: Before any weight is estimated, the plate itself is detected in the photo and used as a built-in ruler. Its pixel diameter is measured and compared against a known real-world plate diameter (26 cm by default) to work out how many pixels correspond to one real cm² in that specific photo. This is done per photo, so the same food item gives a consistent weight estimate whether the picture was taken close up or from farther away. If no plate can be detected, the code falls back to assuming the plate fills roughly 70% of the shorter image dimension.

Weight estimation: Each detected class is handled one of three ways:

Countable items (banana, egg, apple, sausage, and so on): connected component analysis on the mask counts discrete instances, multiplied by an average weight per item.
Area-based items (rice, sauce, mushrooms, and so on): the mask's pixel area is converted to real cm² using the plate-based scale factor, then multiplied by a per-food density factor (grams per cm²) to get weight.
Container-based items (coffee, tea, juice, soup, and so on): these sit in a cup, glass, or bowl, so a flat area estimate would badly undercount them. Instead, the visible mask is treated as the liquid's surface area, multiplied by an assumed container depth to get a volume, then multiplied by the liquid's density to get weight.


Calorie lookup: For each food class, calories per 100g are fetched from the USDA FDC API (first by a known fdc_id, falling back to a name search, and finally to a generic 140 kcal/100g estimate if no match is found). Results are cached to avoid repeat API calls.

Output: A pandas DataFrame lists each detected item, its estimated quantity/weight, kcal per 100g, and total calories, sorted by calorie contribution. A side-by-side plot shows the segmented detections and a pie chart of calorie contribution per item.

Notes and Limitations :

The model could not be trained over higher epochs, so there is a considerable loss associated with it (Colab's T4 GPU limit was hit twice during training).
Weight estimates are still heuristic. Average item weights, area density factors, and container depth assumptions are fixed per food class rather than measured directly, so actual calorie counts will vary with real portion size, plating, and how full a cup or bowl actually is.
The per-photo plate calibration corrects for camera distance and zoom, but it doesn't correct for camera angle. Photos taken at a steep angle rather than straight overhead will still foreshorten items differently depending on how far they sit from the camera, so accuracy is best with a roughly top-down shot.
Container-based items assume a fixed liquid depth for a given food type (for example, coffee assumes about 8 cm), so a half-empty cup and a full one currently get estimated the same way.
Foods without a known FDC ID or USDA match fall back to a generic 140 kcal/100g estimate.
Each user must supply their own USDA API key (free tier).
