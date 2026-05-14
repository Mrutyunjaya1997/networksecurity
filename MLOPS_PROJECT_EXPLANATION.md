# Network Security MLOps Project Explanation

This document is an interview-ready explanation of the Network Security project in this repository. The goal is to help you explain the project end to end: requirement, data flow, development, training pipeline, deployment, monitoring, and future improvements.

## 1. Project Overview

This project is an MLOps-based machine learning application for detecting whether a website or URL pattern is phishing or legitimate using network/security-related features.

The project takes phishing data, stores it in MongoDB, builds a training pipeline, validates the data, transforms missing values, trains multiple classification models, selects the best model, tracks model metrics using MLflow/DagsHub, saves model artifacts, syncs artifacts to AWS S3, and exposes a FastAPI application for training and prediction.

In simple words:

> I built an end-to-end MLOps pipeline for phishing website detection. The system ingests data from MongoDB, validates schema and drift, transforms the data, trains and compares multiple ML models, tracks experiments, stores artifacts, containerizes the application, and deploys it through a CI/CD pipeline using GitHub Actions, Docker, AWS ECR, and an EC2/self-hosted runner.

## 2. Business Requirement

Phishing websites are fake or malicious websites that try to steal sensitive information such as login credentials, banking details, or personal data. A security system should be able to identify risky websites based on measurable URL and website characteristics.

The main requirement of this project is:

- Build a machine learning model that can classify a website as phishing or legitimate.
- Automate the complete ML lifecycle from data ingestion to deployment.
- Make the trained model available through an API so users can upload data and get predictions.
- Track experiments and model performance for reproducibility.
- Store generated artifacts and final models for future use.
- Create a deployment pipeline so new code can be built and deployed automatically.

## 3. Dataset

The dataset used in this project is stored locally in:

```text
Network_Data/phisingData.csv
```

It contains multiple numerical features that describe URL and website behavior. The target column is:

```text
Result
```

Some important features are:

- `having_IP_Address`
- `URL_Length`
- `Shortining_Service`
- `having_At_Symbol`
- `Prefix_Suffix`
- `SSLfinal_State`
- `Domain_registeration_length`
- `Request_URL`
- `URL_of_Anchor`
- `web_traffic`
- `Page_Rank`
- `Google_Index`
- `Statistical_report`

The schema is defined in:

```text
data_schema/schema.yaml
```

This schema acts as the contract for incoming data. It defines the expected columns and their data types.

## 4. High-Level Architecture

The project has these main layers:

```text
Raw CSV data
    |
    v
MongoDB data storage
    |
    v
Data ingestion
    |
    v
Data validation and drift detection
    |
    v
Data transformation
    |
    v
Model training and model selection
    |
    v
MLflow/DagsHub experiment tracking
    |
    v
Saved model artifacts
    |
    v
FastAPI prediction service
    |
    v
Docker + GitHub Actions + AWS ECR + EC2 deployment
```

## 5. Main Project Structure

Important files and folders:

```text
networksecurity/
    components/
        data_ingestion.py
        data_validation.py
        data_transformation.py
        model_trainer.py

    pipeline/
        training_pipeline.py

    entity/
        config_entity.py
        artifact_entity.py

    constant/
        training_pipeline/__init__.py

    utils/
        main_utils/utils.py
        ml_utils/model/estimator.py
        ml_utils/metric/classification_metric.py

    cloud/
        s3_syncer.py

app.py
main.py
push_data.py
Dockerfile
.github/workflows/main.yml
data_schema/schema.yaml
Network_Data/phisingData.csv
final_model/
Artifacts/
```

## 6. End-to-End Pipeline Explanation

### Step 1: Data Collection and Upload to MongoDB

The raw data starts as a CSV file:

```text
Network_Data/phisingData.csv
```

The file `push_data.py` reads this CSV, converts the data into JSON records, and inserts those records into MongoDB.

Important flow:

```text
CSV file -> Pandas DataFrame -> JSON records -> MongoDB collection
```

MongoDB database and collection used by the project:

```text
Database: MJDB
Collection: NetworkData
```

Interview explanation:

> First, I collected the phishing dataset as a CSV file. To make the project closer to a real production setup, I pushed the raw records into MongoDB. This simulates a centralized data source from where the training pipeline can pull fresh data whenever retraining is triggered.

### Step 2: Data Ingestion

Code file:

```text
networksecurity/components/data_ingestion.py
```

The data ingestion component connects to MongoDB, reads all records from the configured collection, converts them into a Pandas DataFrame, removes the MongoDB `_id` column, replaces `"na"` values with `np.nan`, saves the complete dataset into a feature store folder, and splits the data into train and test CSV files.

Artifacts generated:

```text
Artifacts/<timestamp>/data_ingestion/feature_store/phisingData.csv
Artifacts/<timestamp>/data_ingestion/ingested/train.csv
Artifacts/<timestamp>/data_ingestion/ingested/test.csv
```

The train-test split ratio is defined in:

```text
networksecurity/constant/training_pipeline/__init__.py
```

Current value:

```text
test_size = 0.2
```

Interview explanation:

> In the ingestion stage, the pipeline pulls data from MongoDB and creates a reproducible local artifact. I save both the complete ingested data and the train-test split. This is useful because every training run gets timestamped artifacts, so I can later debug or reproduce that run.

### Step 3: Data Validation

Code file:

```text
networksecurity/components/data_validation.py
```

The data validation component checks:

- Whether the input train and test data have the expected columns.
- Whether there is data drift between train and test distributions.

The expected schema is stored in:

```text
data_schema/schema.yaml
```

For drift detection, the project uses the Kolmogorov-Smirnov test:

```text
scipy.stats.ks_2samp
```

The drift report is saved as:

```text
Artifacts/<timestamp>/data_validation/drift_report/report.yaml
```

Interview explanation:

> After ingestion, I validate the data before training. I check the schema to make sure the expected columns are present. I also run a statistical drift check between the train and test distributions using the KS test. If the p-value indicates that distributions are significantly different, I mark drift for that feature and save the report as a YAML artifact.

Why this matters:

- Prevents training on bad or unexpected data.
- Helps catch changes in feature distribution.
- Gives visibility into data quality before model training.

### Step 4: Data Transformation

Code file:

```text
networksecurity/components/data_transformation.py
```

The transformation stage separates input features from the target column `Result`.

It also converts target values:

```text
-1 -> 0
```

This makes the target suitable for binary classification.

Missing values are handled using:

```text
sklearn.impute.KNNImputer
```

The imputer configuration is:

```text
n_neighbors = 3
weights = "uniform"
```

The transformation output is saved as NumPy arrays:

```text
Artifacts/<timestamp>/data_transformation/transformed/train.npy
Artifacts/<timestamp>/data_transformation/transformed/test.npy
```

The preprocessing object is saved as:

```text
Artifacts/<timestamp>/data_transformation/transformed_object/preprocessing.pkl
final_model/preprocessor.pkl
```

Interview explanation:

> In the transformation stage, I separate features and target, convert the target into a binary classification format, and handle missing values using KNN imputation. I save the transformed train and test arrays as `.npy` files and also persist the preprocessing object. Persisting the preprocessor is important because the same transformation must be applied during prediction.

### Step 5: Model Training

Code file:

```text
networksecurity/components/model_trainer.py
```

The model trainer loads the transformed train and test NumPy arrays and trains multiple classification algorithms.

Models used:

- Random Forest
- Decision Tree
- Gradient Boosting
- Logistic Regression
- AdaBoost

Hyperparameter tuning is performed using:

```text
GridSearchCV
```

The helper function is:

```text
networksecurity/utils/main_utils/utils.py
```

After training, the best model is selected and wrapped with the preprocessor using the custom `NetworkModel` class.

Model wrapper:

```text
networksecurity/utils/ml_utils/model/estimator.py
```

The final model artifacts are saved in:

```text
Artifacts/<timestamp>/model_trainer/trained_model/model.pkl
final_model/model.pkl
```

Interview explanation:

> In model training, I compare multiple classification algorithms instead of depending on a single model. I use GridSearchCV for hyperparameter tuning and select the best performing model. Then I combine the preprocessing object and the trained model into a reusable prediction pipeline. This ensures that training-time and inference-time transformations remain consistent.

### Step 6: Model Evaluation

The project calculates classification metrics using:

```text
networksecurity/utils/ml_utils/metric/classification_metric.py
```

Metrics tracked:

- F1 score
- Precision
- Recall

Interview explanation:

> Since this is a phishing detection use case, accuracy alone is not enough. Precision and recall are important because false negatives can be dangerous. If a phishing website is classified as legitimate, that is a serious security risk. F1 score gives a balanced view of precision and recall.

### Step 7: Experiment Tracking with MLflow and DagsHub

The project uses MLflow for experiment tracking and DagsHub as the remote MLflow tracking server.

Tracked information:

- F1 score
- Precision
- Recall
- Model artifact
- Registered model name

Code location:

```text
networksecurity/components/model_trainer.py
```

Interview explanation:

> I integrated MLflow with DagsHub to track experiments remotely. For every training run, I log model metrics such as F1 score, precision, and recall. This helps compare different models and training runs. It also improves reproducibility because I can see which model version performed best and what metrics it produced.

Important production note:

Secrets such as MLflow tracking credentials should be stored in environment variables or secret managers, not directly in source code.

### Step 8: Artifact Management

The project generates timestamped artifacts under:

```text
Artifacts/<timestamp>/
```

Examples:

```text
data_ingestion/
data_validation/
data_transformation/
model_trainer/
```

The final model and preprocessor are stored in:

```text
final_model/
```

The training pipeline also syncs artifacts and final models to AWS S3 using:

```text
networksecurity/cloud/s3_syncer.py
```

S3 bucket configured:

```text
netwworksecuritymj
```

Interview explanation:

> Every training run creates timestamped artifacts. This is important in MLOps because it gives traceability. I can go back and inspect exactly what data, transformation object, drift report, and model were produced in a particular run. I also sync artifacts to S3 so they can be stored outside the local machine and reused later.

### Step 9: Training Pipeline Orchestration

Code file:

```text
networksecurity/pipeline/training_pipeline.py
```

This file orchestrates the full workflow:

```text
Data ingestion
    -> Data validation
    -> Data transformation
    -> Model training
    -> Sync artifacts to S3
    -> Sync final model to S3
```

Main method:

```python
run_pipeline()
```

Interview explanation:

> I created a pipeline class that controls the complete training workflow. Each stage returns an artifact object, and that artifact becomes the input for the next stage. This makes the pipeline modular, testable, and easier to debug.

Why artifact objects are useful:

- They clearly define what each stage produces.
- They avoid hardcoding too many paths across components.
- They make the pipeline easier to extend.

### Step 10: FastAPI Application

Code file:

```text
app.py
```

The application exposes two main endpoints:

```text
GET /train
POST /predict
```

`GET /train`:

- Starts the complete training pipeline.
- Ingests data, validates data, transforms data, trains model, and syncs artifacts.

`POST /predict`:

- Accepts a CSV file upload.
- Loads the saved preprocessor and model from `final_model/`.
- Applies transformation.
- Generates predictions.
- Saves predictions to `prediction_output/output.csv`.
- Displays results in an HTML table using `templates/table.html`.

Interview explanation:

> I used FastAPI to expose the ML system as a service. The `/train` endpoint can trigger retraining, and the `/predict` endpoint accepts a CSV file and returns predictions. For prediction, I load the saved preprocessor and model, apply the same transformation as training, and return the prediction results.

### Step 11: Containerization with Docker

Docker file:

```text
Dockerfile
```

The Docker image:

- Uses Python 3.10 slim image.
- Copies the project into the container.
- Installs AWS CLI.
- Installs Python dependencies from `requirements.txt`.
- Starts the FastAPI app using `python3 app.py`.

Interview explanation:

> I containerized the application using Docker so that it can run consistently across environments. The Docker image contains the application code, dependencies, and AWS CLI for syncing artifacts. This makes deployment easier and reduces environment mismatch issues.

### Step 12: CI/CD with GitHub Actions, AWS ECR, and EC2

Workflow file:

```text
.github/workflows/main.yml
```

The CI/CD pipeline has three jobs:

1. Continuous Integration
2. Continuous Delivery
3. Continuous Deployment

Current CI step:

- Checks out the code.
- Placeholder lint command.
- Placeholder test command.

Continuous Delivery:

- Builds Docker image.
- Logs in to AWS ECR.
- Pushes Docker image to ECR.

Continuous Deployment:

- Runs on a self-hosted runner.
- Pulls latest image from ECR.
- Runs the container on port `8080`.
- Cleans old Docker images.

Interview explanation:

> I configured GitHub Actions to automate build and deployment. On every push to the main branch, the workflow builds a Docker image, pushes it to AWS ECR, and deploys it on a self-hosted runner, which can be an EC2 machine. This gives the project a basic CI/CD pipeline.

## 7. Configuration Management

The project stores pipeline constants in:

```text
networksecurity/constant/training_pipeline/__init__.py
```

Configuration classes are defined in:

```text
networksecurity/entity/config_entity.py
```

Artifact classes are defined in:

```text
networksecurity/entity/artifact_entity.py
```

This structure separates:

- Constants
- Configuration
- Runtime artifacts
- Component logic

Interview explanation:

> I separated constants, configuration classes, and artifact classes. This improves maintainability because each pipeline stage receives its own config object and returns a clear artifact object. This design is inspired by production ML pipelines where each stage has defined inputs and outputs.

## 8. Logging and Exception Handling

Custom exception handling is implemented in:

```text
networksecurity/exception/exception.py
```

Logging is implemented in:

```text
networksecurity/logging/logger.py
```

Interview explanation:

> I added custom exception handling and logging so that failures are easier to debug. In an ML pipeline, failures can happen at many stages such as database connection, schema validation, transformation, model training, or artifact saving. Centralized logging helps identify where and why the pipeline failed.

## 9. Monitoring in This Project

The project currently has basic monitoring and observability through:

- Data drift report using KS test.
- MLflow experiment tracking.
- Logged training metrics.
- Saved artifacts for traceability.
- Application logs.

How to explain monitoring:

> For monitoring, I currently track model metrics through MLflow and data drift through the validation step. The drift report tells me whether the incoming data distribution has changed. MLflow helps me compare model versions and monitor training performance across experiments. Application logs help debug API or pipeline failures.

## 10. Production Monitoring Improvements

For a more production-ready version, I would add:

- Model performance monitoring on real prediction data.
- Data quality checks on incoming prediction CSV files.
- Feature drift monitoring between training data and production data.
- Prediction drift monitoring.
- API latency and error-rate monitoring.
- Container health checks.
- Automated alerts using CloudWatch, Prometheus, Grafana, or another monitoring tool.
- Scheduled retraining when drift crosses a threshold.
- Model registry approval workflow before production deployment.

Interview explanation:

> The current project has the foundation for monitoring, but in production I would extend it with real-time monitoring. I would track API latency, failed requests, input data quality, feature drift, prediction drift, and post-deployment model performance. If drift or performance degradation crosses a threshold, I would trigger retraining or alert the team.

## 11. Complete Interview Talk Track

You can use this as your main explanation:

> This is an end-to-end MLOps project for phishing website detection. The goal is to classify websites as phishing or legitimate based on URL and website-related features.
>
> The raw dataset is first uploaded to MongoDB using a data push script. The training pipeline starts by ingesting data from MongoDB, converting it into a DataFrame, removing unnecessary columns, handling missing values as `np.nan`, and splitting it into train and test data.
>
> After ingestion, the data validation stage checks whether the data follows the expected schema. It also performs data drift detection using the Kolmogorov-Smirnov test and saves the drift report as a YAML artifact.
>
> In the transformation stage, I separate features and target, convert the target into binary format, and use KNNImputer to handle missing values. The transformed train and test arrays are saved, and the preprocessing object is also saved so that the same transformation can be applied during inference.
>
> In the model training stage, I train multiple classification models such as Random Forest, Decision Tree, Gradient Boosting, Logistic Regression, and AdaBoost. I use GridSearchCV for hyperparameter tuning and select the best model. I evaluate the model using F1 score, precision, and recall, which are more meaningful than only accuracy for a security classification use case.
>
> I use MLflow with DagsHub for experiment tracking. Metrics and model artifacts are logged so I can compare different runs and maintain reproducibility.
>
> The pipeline saves artifacts in timestamped folders and syncs them to AWS S3. The final model and preprocessor are saved in the `final_model` folder.
>
> For serving, I built a FastAPI application. It has a `/train` endpoint to trigger the training pipeline and a `/predict` endpoint where users can upload a CSV file and receive prediction results.
>
> For deployment, I containerized the application using Docker and created a GitHub Actions workflow. The workflow builds the Docker image, pushes it to AWS ECR, and deploys it using a self-hosted runner, which can run on EC2.
>
> Overall, this project covers data ingestion, validation, transformation, model training, experiment tracking, artifact management, API serving, Dockerization, CI/CD, and basic monitoring.

## 12. What Makes This an MLOps Project

This is not just a machine learning notebook. It includes important MLOps practices:

- Modular pipeline components.
- Data ingestion from a database.
- Schema validation.
- Data drift detection.
- Reproducible artifacts.
- Experiment tracking with MLflow.
- Model and preprocessor persistence.
- API-based serving.
- Docker containerization.
- CI/CD pipeline.
- AWS ECR integration.
- S3 artifact storage.
- Centralized logging and exception handling.

## 13. Things to Mention If Asked About Challenges

### Challenge 1: Data Quality

Data can have missing values, unexpected columns, or changed distributions.

Solution used:

- Schema validation.
- KNN imputation.
- Drift detection report.

### Challenge 2: Training and Inference Consistency

The same preprocessing must be used during training and prediction.

Solution used:

- Save the preprocessing object as `preprocessor.pkl`.
- Load the same preprocessor during prediction.

### Challenge 3: Experiment Reproducibility

Different models and parameters can produce different results.

Solution used:

- MLflow tracking.
- Timestamped artifacts.
- Saved model objects.

### Challenge 4: Deployment Consistency

The app should run the same way locally and in production.

Solution used:

- Docker containerization.
- Dependency management through `requirements.txt`.

### Challenge 5: Automation

Manual deployment is slow and error-prone.

Solution used:

- GitHub Actions CI/CD.
- AWS ECR image push.
- Self-hosted deployment runner.

## 14. Important Improvements You Can Mention

These are useful interview points because they show that you understand production readiness.

### Security Improvements

- Move hardcoded MLflow/DagsHub credentials out of code.
- Use environment variables, GitHub Secrets, AWS Secrets Manager, or Parameter Store.
- Rotate any exposed token.
- Avoid printing database URLs in logs.
- Add authentication to `/train` endpoint so anyone cannot trigger retraining.

### Code Quality Improvements

- Add real unit tests in the CI pipeline.
- Add linting using tools like `flake8`, `ruff`, or `pylint`.
- Add type checking using `mypy`.
- Add pre-commit hooks.
- Improve validation logic to check actual schema columns and data types.
- Use classification metrics for model selection instead of regression-style `r2_score`.

### Model Improvements

- Use cross-validation with stratified splits.
- Track confusion matrix and ROC-AUC.
- Tune threshold depending on phishing risk.
- Add model explainability using SHAP or feature importance.
- Add model versioning and stage transitions in MLflow Model Registry.

### MLOps Improvements

- Add scheduled retraining.
- Add automated retraining when drift is detected.
- Add model approval before production deployment.
- Add separate dev, staging, and production environments.
- Store data versions using DVC or lakehouse/versioned storage.
- Use orchestration tools such as Airflow, Prefect, or Kubeflow.

### Deployment Improvements

- Add Docker health check.
- Add proper process server configuration.
- Deploy behind a reverse proxy or load balancer.
- Add API authentication.
- Add request validation for uploaded prediction files.
- Deploy on ECS, EKS, or another managed service for better scalability.

### Monitoring Improvements

- Monitor API latency, request count, and errors.
- Monitor input data drift in production.
- Monitor prediction distribution.
- Monitor model performance when actual labels become available.
- Add alerting through CloudWatch, Prometheus, Grafana, or similar tools.

## 15. Short Version for Interview Introduction

Use this when the interviewer says, "Tell me about your project."

> I worked on a Network Security MLOps project for phishing detection. The project uses a phishing dataset with URL and website-related features and builds an automated ML pipeline. Data is stored in MongoDB, ingested into the pipeline, validated using schema checks and drift detection, transformed using KNN imputation, and used to train multiple classification models. I track experiments with MLflow and DagsHub, save artifacts locally and to S3, expose training and prediction through FastAPI, containerize the app using Docker, and deploy it using GitHub Actions, AWS ECR, and a self-hosted runner. The project demonstrates the full ML lifecycle from data ingestion to deployment and monitoring foundations.

## 16. Very Short Version

> This is an end-to-end MLOps pipeline for phishing website detection. It includes MongoDB data ingestion, schema validation, drift detection, preprocessing, model training, MLflow experiment tracking, S3 artifact storage, FastAPI serving, Docker containerization, and CI/CD deployment through GitHub Actions and AWS ECR.

## 17. Possible Interview Questions and Answers

### Q1. Why did you use MongoDB?

MongoDB acts as the central data source. Instead of reading only from a local CSV, the pipeline reads from a database, which is closer to real-world production systems where data is stored centrally and updated over time.

### Q2. Why did you use data validation?

Data validation prevents bad data from entering the training process. It checks whether expected columns are present and whether train and test distributions have drifted.

### Q3. What is data drift?

Data drift means the statistical distribution of input data changes over time. If production data becomes different from training data, model performance can degrade. In this project, I use the KS test to detect distribution differences.

### Q4. Why did you save the preprocessor?

The model was trained on transformed data, so prediction data must go through the same transformation. Saving the preprocessor ensures consistency between training and inference.

### Q5. Why did you use MLflow?

MLflow helps track experiments, metrics, parameters, and models. It makes it easier to compare training runs and reproduce results.

### Q6. Why use Docker?

Docker packages the code, dependencies, and runtime into a single image. This reduces environment issues and makes deployment consistent.

### Q7. What is the role of GitHub Actions?

GitHub Actions automates CI/CD. It checks the code, builds the Docker image, pushes it to AWS ECR, and deploys it on the target server.

### Q8. How would you improve this project for production?

I would improve secrets management, add real unit tests, add stronger monitoring, use proper model registry promotion, add authentication, improve schema validation, use classification metrics for model selection, and add automated retraining based on drift or performance degradation.

## 18. Honest Notes About Current Code

These are not failures. They are good points to mention as improvements if the interviewer asks about production readiness.

- The CI workflow currently has placeholder lint and test commands. It should be replaced with real tests.
- Some credentials are hardcoded in `model_trainer.py`. They should be moved to environment variables or a secret manager.
- `app.py` uses `MONGODB_URL`, while ingestion scripts use `MONGO_DB_URL`. The environment variable naming should be standardized.
- Model selection currently uses `r2_score`, which is not ideal for classification. It should use F1 score, recall, precision, ROC-AUC, or a business-specific cost function.
- Schema validation should check actual column names and data types from `schema.yaml`.
- The `/train` endpoint should be protected because retraining should not be publicly accessible.
- The S3 sync uses `os.system`; using `boto3` or safer subprocess handling would be better.

## 19. Best Closing Statement

> This project helped me understand that MLOps is not only about training a model. It is about building a reliable system around the model: data ingestion, validation, reproducibility, tracking, artifact management, deployment, monitoring, and continuous improvement. The model is only one part of the system; the pipeline and operational reliability are equally important.

