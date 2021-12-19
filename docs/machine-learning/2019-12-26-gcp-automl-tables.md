---
layout: post
title: GCP AutoML Tables
parent: Machine Learning
tags: [Machine Learning, AutoML, Supervised Learning]
nav_order: 1
---

# Google AutoML Tables
Google Cloud 에서 제공하는 기계학습 서비스 중 하나인 [AutoML Tables](https://cloud.google.com/automl-tables/)에 대한 간략한 내용 및 유의사항을 정리한다.

<br/>

## AutoML Tables
AutoML Tables은 구조화된 데이터(Structured data)를 기반으로 자동화된 **지도학습(Supervised learning)** 및 모델 배포를 지원하는 서비스이다. 
기존에는 학습 단계에서 모델 성능 개선을 위해 다양한 노력들이 필요했는데(알고리즘 선정, 구현, 파라미터 튜닝 등), AutoML은 스스로 다양한 알고리즘들과 파라미터들을 조절해가면서 최적의 모델을 찾아주는 기능을 제공한다.

쉽게 말해서, 기계학습에 대한 지식이나 큰 노력 없이 비즈니스에 필요한 학습모델을 찾아낼 수 있도록 도와주는 서비스라고 보면 되겠다.

현재는 베타 버전으로, 프로덕션 적용을 위해서는 어느정도 직접 고도화하는 노력은 필요해 보인다.

> _At beta, products or features are ready for broader customer testing and use. Betas are often publicly announced. There are no SLAs or technical support obligations in a beta release unless otherwise specified in product terms or the terms of a particular beta program. The average beta phase lasts about six months._

<br/>

## Features 
### Model quality
Google 에서 보유한 다양한 State-of-the-art 알고리즘들을 기반으로 최적의 모델 성능을 제공한다. 일단 구글에서 구현했으니 충분히(?) 믿고 사용하면 되겠다는 생각이 든다.

### Feature engineering
데이터에 대한 통계 바탕으로 손쉽게 Feature 엔지니어링을 할 수 있도록 지원한다.
오해하면 안되는게 이를 자동으로 해준다는 것은 아니다. 제공되는 통계를 바탕으로 어느 정도는 직접 Feature를 선택하는 노력이 필요하다.

### Model deployment & serving
Google이 보유한 확장성있는 환경에서의 모델 배포 및 serving을 지원하므로 프로덕션 환경에 빠르게 적용이 가능하다.
> _하지만, 현재 베타 버전을 기준으로 GCP에서 직접 제공하는 Online Prediction을 프로덕션에 적용하는것은 어려워보인다(뒤에서 자세히 설명)._

<br/>

## Preparing training data
당연한 이야기지만 학습을 위해서는 먼저 데이터를 준비하는 과정이 필요하다. 아래는 AutoML Tables 에서 학습 데이터의 준비를 위해 어떤 과정들이 필요한지에 대해 설명한다.

### 1) 문제 타입 결정
풀고자 하는 문제의 타입이 **Classification**인지, **Regression**인지 결정한다.
- Classification: Categorical value를 예측
- Regression: Numeric value를 예측

문제 타입은 추후 AutoML Tables의 파라미터로 입력된다.

> _Classification 문제의 경우 target 개수 범위는 [2, 500] 이어야한다._

### 2) 학습용으로 사용할 데이터 결정
AutoML Tables은 아래와 같은 학습 데이터 제약사항을 가지므로, 데이터 준비 시 이를 유의해야 한다.
- It must be 100 GB or smaller.
- The value you want to predict (your target column) must be included.
- There must be at least two and no more than 1,000 columns.  
- There must be at least 1,000 and no more than 100,000,000 rows.

> _보유하고 있는 데이터가 너무 큰 경우, 중요한 Feature만 선택하거나 영향력이 큰 Row만 선택(ex. 보다 최신의 데이터)하는 전략이 필요해 보인다._

### 3) 추가적인 속성 정의
보다 정교한 학습을 위해 아래의 메타 속성들을 데이터 Column에 추가해서 사용할 수 있다.

**Data Split Column**은 각 row가 어떤 용도로 사용될지 지정하는 컬럼으로, 아래와 같은 조합으로 사용이 가능하다.
- All of TRAIN, VALIDATE, and TEST
- Only TEST and UNASSIGNED

> _UNASSIGNED의 경우, AutoML Tables가 자동으로 training 와 validation set으로 나눈다._

**Time Column**은 Timestamp 형식의 값을 지니는 컬럼으로, 해당 row가 언제 발생한 데이터인지를 지정하기 위한 용도이다. 이 컬럼을 지정한 경우 AutoML Tables는 가장 오래된 80%는 training, 다음 10%는 validation, 마지막 최신 10%는 test 용도로 사용한다.

**Weight Column**은 특정 row에 가중치를 부여하기 위한 컬럼이다. [0, 10000] 범위의 값이 지정 가능하며, 높을수록 더 많은 가중치를 부여받게 된다.

### 4) 데이터 Import source 준비
AutoML Tables로 학습 데이터를 전달하기 위한 방법은 두 가지가 존재한다. 아래의 방법 중 보다 본인에게 적절한 방법을 선택하고 데이터를 미리 준비한다.
  - [Using BigQuery](https://cloud.google.com/automl-tables/docs/prepare#bq)
  - [Using CSV files](https://cloud.google.com/automl-tables/docs/prepare#csv)

BigQuery 테이블을 Import 하는 경우, 별도의 데이터 스키마를 지정할 필요가 없기 때문에 편리하다.

> _단, 현재 버전 기준으로 BigQuery dataset이 US 또는 EU 리전에 존재해야 한다._

<br/>

## Training
학습 데이터가 잘 준비되었으면 이제 실제 모델을 학습시킬 수 있다.

앞서 설명처럼 학습 관련 작업은 크게 할 것이 없다. [Preparing training data](#preparing-training-data) 단계에서 결정된 사항들이나 간단한 필수 설정값(ex. 학습 시간, target column)들을 AutoML Tables에서 제공하는 인터페이스([GCP Console](https://cloud.google.com/automl-tables/docs/train#automl-tables-model-create-console) 또는[Restful API](https://cloud.google.com/automl-tables/docs/train#automl-tables-model-create-cli-curl))를 통해 지정하고 학습을 시작하면 된다. 자세한 사용 방법은 링크된 페이지를 참고한다.

### 참고사항
**Training budget 설정**

AutoML Tables의 [과금 정책](https://cloud.google.com/automl-tables/pricing)은 Compuing 시간 당 비용이다. 따라서 시간(Hour)값의 예산을 설정해야 하는데, 이를 모두 사용하기 전에 성능이 수렴했다고 판단되면 미리 학습이 종료되기도 한다(Ealry Stopping). 물론 이 옵션을 끌수도 있다.

**최적화 목표 (Optimization Objective) 설정**

문제의 종류에 따라 다양한 최적화 목표를 제공한다 ([참고](https://cloud.google.com/automl-tables/docs/train#opt-obj)

풀고자 하는 문제의 종류 및 성격에 따라 적절한 최적화 목료를 지정해서 사용하면 된다.

### 학습 시 유의사항
각 컬럼의 타입이 적절히 지정되었는지 확인이 필요하다.
- 자연어가 가능한 필드는 Text 필드로 지정되어야 한다.
- 실제는 Categorical 필드인데 Numeric 으로 지정된 필드는 없는지 확인한다(ex. Red=1, Yellow=2).

아래와 같이 학습을 방해하는 컬럼은 학습에서 배제하는것이 필요하다.
- 의도하지 않은 Missing value 나 Invalid value 가 많은 컬럼
- ID와 같이 거의 대부분의 row가 서로 다른 값을 지니는 컬럼
- Target 컬럼과 연관성이 높은데, 사실은 Target 에 의해 유도되는 컬럼

<br/>

## Evaluation
모델 학습을 완료했으면 이제 해당 모델의 성능을 평가해서 적용 유무를 결정해야한다.
AutoML Tables 에서는 아래와 같은 평가 지표들 제공하며, 이를 활용해서 모델 성능이 비즈니스 요구사항에 부합하는지 평가가 가능하다.

### Classification 모델인 경우
문제 타입이 Classification인 경우, 아래와 같은 지표들을 활용할 수 있다.
- AUC PR (Area Under the Precision-Recall Curve)
- AUC ROC (Area Under the Receiver Operating Characteristic Curve)
- Accuracy
- Log loss
- F1 score
- Precision
- Recall
- False positive rate

### Regression 모델인 경우
- MAE (Mean Absolute Error)
- RMSE (Root Mean Square Error)
- RMSLE (Root Mean Squared Logarithmic Error)
- R^2 (R Squared)
- MAPE (Mean Absolute Percentage Error)

또한, 문제 타입에 관계없이 **Feature importance**를 제공하므로, 무의미한 Feature가 사용되지는 않았는지 추가적으로 확인할 수 있다.

AutoML Tables 에서는 이러한 지표들을 획득하기 위한 인터페이스로 [GCP Console](https://cloud.google.com/automl-tables/docs/evaluate#automl-tables-list-model-evaluations-console)과 [Restful API](https://cloud.google.com/automl-tables/docs/evaluate#automl-tables-list-model-evaluations-cli-curl)를 제공한다.

## References
- [Google AutoML Tables](https://cloud.google.com/automl-tables/docs/)