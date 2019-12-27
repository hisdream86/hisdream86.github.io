---
layout: default
category: [ml]
title: GCP AutoML Tables
tags: [gcp, automl, tables, supervised learning]
---

# Google AutoML Tables
Google Cloud 에서 제공하는 기계학습 서비스 중 하나인 [AutoML Tables](https://cloud.google.com/automl-tables/)에 대한 간략한 내용 및 유의사항을 정리한다.

<br/>

## AutoML Tables
AutoML Tables은 구조화된 데이터(Structured data)를 기반으로 자동화된 **지도학습(Supervised learning)** 및 모델 배포를 지원하는 서비스이다. 
기존에는 학습 단계에서 모델 성능 개선을 위해 다양한 노력들이 필요했는데(알고리즘 선정 및 구현, 파라미터 튜닝 등), AutoML은 스스로 다양한 알고리즘들과 파라미터들을 조절해가면서 최적의 모델을 찾아주는 기능을 제공한다.

쉽게 말하면, 기계학습에 대한 지식 없이도 비즈니스에 필요한 최적 학습모델을 찾아낼 수 있도록 지원하는 서비스라고 보면 되겠다.

현재는 베타 버전으로, 프로덕션에 적용을 위해서는 어느정도는 직접 고도화하는 노력이 필요해 보인다.

<br/>

## Features 
#### Model quality
Google 에서 보유한 다양한 State-of-the art 알고리즘들을 기반으로 최적의 모델 성능을 제공한다.

#### Feature engineering
데이터에 대한 통계 바탕으로 손쉽게 Feature 엔지니어링을 할 수 있도록 지원한다.
오해하면 안되는게 있는데, Feature 엔지니어링을 자동으로 해준다는 것은 아니다. 제공되는 통계를 바탕으로 어느 정도는 직접 Feature를 선택하는 노력이 필요하다.

#### Model deployment
Google 인프라를 통해 모델을 serving 하므로 프로덕션 환경에 쉽게 적용이 가능하다
> 하지만, 현재 베타 버전을 기준으로 GCP에서 직접 제공하는 Online Prediction을 프로덕션에 적용하는것은 어려워보인다(뒤에서 자세히 설명).

<br/>

## Preparing training data
### 문제 타입 결정
풀고자 하는 문제의 타입이 **Classification**인지, **Regression**인지 결정한다.
- Classification: Categorical value를 예측
- Regression: Numeric value를 예측

문제 타입은 추후 AutoML Tables의 파라미터로 입력된다.

> Classification 문제의 경우 target 개수 범위는 [2, 500] 이어야한다.

### 학습 데이터에 포함시킬 내용 결정
AutoML Tables은 아래와 같은 학습 데이터 제약사항을 가지므로, 데이터 준비 시 이를 유의해야 한다.
- It must be 100 GB or smaller.
- The value you want to predict (your target column) must be included.
- There must be at least two and no more than 1,000 columns.  
- There must be at least 1,000 and no more than 100,000,000 rows.

> 보유하고 있는 데이터가 너무 큰 경우, 중요한 Feature만 선택하거나 영향력이 큰 Row만 선택(ex. 보다 최신의 데이터)하는 전략이 필요해 보인다.

### 추가적인 속성 정의
보다 정교한 학습을 위해 아래의 메타 속성들을 데이터 Column에 추가해서 사용할 수 있다.

**Data Split Column**은 각 row가 어떤 용도로 사용될지 지정하는 Column으로, 아래와 같은 조합으로 사용이 가능하다.
- All of TRAIN, VALIDATE, and TEST
- Only TEST and UNASSIGNED

> UNASSIGNED의 경우, AutoML Tables가 자동으로 training 와 validation set으로 나눈다.

**Time Column**은 Timestamp 형식의 값을 지니는 Column으로, 해당 row가 언제 발생한 데이터인지를 지정하기 위한 용도이다. 이 Column을 지정한 경우 AutoML Tables는 가장 오래된 80%는 training, 다음 10%는 validation, 마지막 최신 10%는 test 용도로 사용한다.

**Weight Column**은 특정 row에 가중치를 부여하기 위한 Column이다. [0, 10000] 범위의 값이 지정 가능하며, 높을수록 더 많은 가중치를 부여받게 된다.

### 데이터 Import source 결정
AutoML Tables로 학습 데이터를 전달하기 위한 방법은 두 가지가 존재한다.
- [Using BigQuery](https://cloud.google.com/automl-tables/docs/prepare#bq)
- [Using CSV files](https://cloud.google.com/automl-tables/docs/prepare#csv)

BigQuery 테이블을 Import 하는 경우, 별도의 데이터 스키마를 지정할 필요가 없기 때문에 편리하다.

> 유의사항: 현재 버전 기준으로 BigQuery dataset이 US 또는 EU 리전에 존재해야 한다(아마 서비스와 동일 리전에 존재해야 하는듯?).

<br/>

## Training models
TBD