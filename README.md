## CIFAR_100

### 모델 5개를 앙상블하여 정확도 **0.8199** 달성

### 100가지 이미지들을 분류하는 Image Classification 모델 개발.


[시간순으로 진행사항 노션정리](https://hail-gray-c42.notion.site/350cbe8e65944ea3bbe5d10fef032c3d)

---
### Introduction
- 2012년 ILSVRC에서 종전의 기록과 당해년도 대비 압도적인 정확도로 1등을 했던 AlexNet의 창시자 Alex krizhevsky가 수집하여 만든 데이터 셋인 CIFAR-10, CIFAR-100 중에 100개의 클래스를 가진 CIFAR-100 데이터셋을 사용하였다
- CIFAR-10에 비해 클래스는 훨씬 더 많으므로 좀더 난이도가 있다고 할 수 있다
- **프로젝트의 목표는 직접 짠 베이스라인으로 정확도 0.65 이상, 전이학습과 사용할 수 있는 모든 기법을 적용해 정확도 0.78 이상 넘기는것.**
- 정말 기본적인 CNN부터 시작하였다

<br/>

### Data
- CIFAR-10/100 공식 사이트 : http://www.cs.utoronto.ca/~kriz/cifar.html
- CIFAR-100은 20개의 슈퍼클래스, 100개의 클래스로 각 클래스당 훈련용 500장, 테스트용 100장으로 총 60000장의 이미지 데이터셋이다
- train image : 50000장 / test image : 10000장
- 이미지의 shape는 32x32x3
- 라벨링 수와 클래스 이름 : https://huggingface.co/datasets/cifar100 참고

### Methods
#### 장문 주의..

- Progress & Result
  - 베이스라인 짜기
    - Simple CNN 만들고 성능확인 
    - [Conv-Pool-Conv-Pool-Conv-Dense-Dense]의 아주 심플한 CNN에서 0.3대의 성능이 나옴. 성능향상을 위해 층을 더 쌓고자 CNN 논문들을 찾아봄
    - 기초 CNN 모델 LeNet-5, AlexNet 구현 및 사용된 기법들을 가져와 베이스라인에 적용
    - => BatchNormalization층 추가, MaxPooling시 Overlapping 적용, AlexNet 구조 사용시 미미한 성능 향상. Conv/Dense의 노드수, 풀링 필터수를 조정해보며 성능을 확인함
    - Conv-Bn-Pool로 이루어진 블럭을 하나 추가 => 0.3초반에서 0.35중반으로 성능 향상. 층이 너무 얇아서 성능이 안나온다고 생각하여 더 깊게 쌓았다
    - 하지만 깊게 쌓을수록 성능이 올라가지 않았으므로 너무 깊지않게 Conv블럭을 쌓아서 0.42 달성. 심한 과적합 문제 발생
    - 과적합 해결을 위해 Dropout층 추가. 0.42 -> 0.46. 과적합도 어느정도 억제하면서 성능향상까지 큰 효과를 보았다
    - epoch수가 100으로는 부족하여 200을 주고 EarlyStopping 적용. 0.46 -> 0.48
    - Conv를 2층 연속으로 주었을땐 0.48 괜찮게 나오다가 Conv의 필터수를 2배로 늘렸더니 0.46으로 성능이 떨어짐. 
    - 필터수를 크게 가져가는 것보단 좀 작더라도 층을 더 많이 거치는 것이 성능에 좋은가 라는 생각을 갖게됨
    - DropOut 위치 변경. 기존에는 Conv-Bn-Act-Dropout-Pool 이였다면 이번에는 Conv-Bn-Act-Pool-Dropout, 풀링과 자리를 바꿈. 0.46 -> 0.52 빼엠. 큰 성능향상... 
    - 이에 대한 정확한 성능 향상 원인은 찾지 못했으나 DropOut이 일정 비율의 노드를 비활성화하는 것이고 Pooling은 일종의 정보압축이니까 Dropout이 Pooling 앞에 있다면 정보 압축 전에 일정 비율의 정보를 누락하고 압축하기에 그 과정에서 정보 손실이 일어나는 것이 아닐까라는 생각을 하였다. 
    - 그외에 Bn, Act의 층 위치 변경등을 해보며 성능확인
    - VGG16, 19 구현 및 공부. Conv 한블럭을 제거해봤더니 -> 0.56. 빼엠. 층을 너무 깊게 쌓았나 보다
    - 완전연결층을 제거해보고, 필터수, Dropout 비율 조정하며 실험. 0.57 -> 0.58 -> 0.59 이얏호
    - 위에서 조정하며 만들어온 베이스라인에 학습률을 Adam, 0.0005를 줘봤다 => 0.62!!  0.6 돌파! 0.65가 코앞이다
    - 이제 뭘 더 해볼까 생각하다가 ResNet으로 0.8을 찍은걸 리더보드에서 보고 ResNet을 공부하고 구현해보았다
    - ResNet의 가장 큰 특징, ShortCut-Connection을 베이스라인에 적용. 하지만 0.5후반 0.6초반에서 벗어나지 못했다...
    - 멘토님의 피드백을 받고 필터수에 대한 실험을 해보았다. 이미지 크기랑 채널, 필터 사이즈는 비슷하게 흘러가야 성능이 좋아진다. ok 확인
    - Conv층 필터수를 8부터 시작, 최적의 필터수를 찾는 실험을 시작함. 다 언급하기엔 너무 많아서 노션 참고
    - 실험을 하면서 필터수가 이미지 사이즈에 비슷하게 갈경우 과적합이 적은대신 성능향상이 낮다는 문제가 발생. 
    - 그럼 필터수를 조금 높게 가져가고 층도 깊게 가져가는데 과적합을 해결하면??? 성능이 좋아지지 않을까?? ok. Augmentation으로 해결해보자
    - ImageDataGenerator을 이용한 Online Augmenation으로 학습, 베이스라인 0.65, 0.67 달성. 무야호
    - 온라인으로 증강도 좋지만 직접 이미지를 만들어 저장하여 증강한 데이터셋을 만들어봄. OpenCV를 통한 Offline Augmentation도 진행
  - 전이학습(Transfer Learning)
    - 데이터 이미지는 32x32x3, keras에서 지원하는 pretrained model들은 224, 240 등 큰 이미지들이므로 이미지 리사이즈가 필요. 하지만 이미지 한장씩 리사이즈하면 연산이 너무 커져서 코랩의 렘이 터진다. 따라서 배치단위로 전처리를 하도록 데이터 API, tf.Dataset를 이용해 데이터셋을 만들어주었다
    - 모델은 동결하고 분류기만 학습하였다.
     <br/>
     
     - model|accuracy|
       |-------|-------|
       |Xception| 0.6456 |
       |MobileNetV2 | 0.5984 |
       |InceptionV3 | 0.6060 |
       |ResNet50 | 0.6693 |
       |ResNet101 | 0.6853 |
       |ResNet152 | 0.7189 |
       |DenseNet121 | 0.6820 |
       |EfficientNetB0 | 0.7252 |
       |EfficientNetB1 | 0.7430 |
       
     - GPU 할당 아웃으로 돌리다가 중간에 멈춰서 성능을 확인하지 못한 모델도 많다...
   - Online Augmentation + Fine-Tune
     - ImageDataGenerator로 75000장 6가지 이미지 변형을 랜덤으로 적용하여 증강시키고 사전학습된 모델의 일부층 동결을 해제하여 Fine-tune을 진행하였다
     
     - model|accuracy|
       |-------|-------|
       |EfficientNetB0 | 0.7449 |
       |EfficientNetB1 | 0.7652, 0.7600 |
       |EfficientNetB2 | 0.7480 |
       |ResNet152 | 0.7395 |
       
     - Augmentation에 큰 이미지로 리사이즈, 큰 모델들에 fine-tune으로 학습해야할 파라미터 증가까지. 코랩의 GPU 할당제한으로 끝까지 학습시킨 모델들은 없고 중간에 ModelCheckPoint로 저장한 파일을 불러와서 성능을 확인하였다.

  - Model Ensemble
    - 위에서 얻은 모델 5개를 앙상블하여 **0.8199**를 달성하였다
    - 정밀도 : 0.8223 / 재현율 : 0.8199 / f1_score : 0.8211
    - Confusion_matrix, ROC-curve는 라벨이 100개이기에 다 크게 나와서 주피터 파일 참고.

       
       
### Review

- 아쉬운점
  - 1. 시간분배의 아쉬움. 
베이스라인을 만드는데 시간을 많이 투자함. 내가 기초가 제대로 되어있었다면 베이스라인은 금방 만들고 성능 향상 기법들을 더 많이 적용했을 것이다. 그래도 이번기회에 직접 쌓아보고 조건을 걸어서 성능을 비교하면서 실험해본 경험은 도움이 되었다
  - 2. 전이학습시 일괄적인 처리 함수가 아닌 모델에 정해져있는 메서드를 사용한점. 
keras.applications을 통해 전이학습할때 가이드를 보면 keras.applications.[model name].preprocess_input으로 모델에 맞는 처리를 해주라는 문장이 있다. 
사실 0\~1, -1\~1 등 이미지 스케일링을 한 임의의 전처리 함수 사용시 전이학습때 성능이 나오지 않는 문제가 있었다… keras.applications.[model name].preprocess_input을 사용하면 성능이 잘나오는데 내가 짠 전처리함수를 쓰면 성능이 안나온다. 내가 어딜 잘못했나?? 
더 복잡한건 같은 모델의 preprocess_input이 아니면 성능이 안나온다.  물론 모델에 최적화되어있는 전처리니 성능을 최대한 높이려면 이걸 쓰는 게 맞긴한데… 모델마다 처리 결과물은 0\~1 이거나 -1\~1 이기도 하고, ResNet의 경우는 255나 127.5로 나누지않고 제로 센터링 해버린다.
EfficientNet의 preprocess_input을 사용해서 ResNet에 테스트나 학습하면 성능이 안좋다. 거기다 앙상블 시에도 모델마다 다른 함수를 적용해 모델마다의 test_dataset을 만들어줘야 했다… 이게 맞나?? 
  - 3. 직접 짠 베이스 라인으로 0.7 넘길때까지 실험해보지 못한것. 앙상블을 안해본 것
이미지 증강을 통해 0.65, 0.67까지 달성을 하였다. 하지만 최적의 필터수, 블럭 조합 찾겠다고 계정 5~6개씩 돌리면서 모델 파일을 저장안 한것이 너무 아쉬웠다. 0.6 넘는 모델들로 10몇개 싹 모아서 앙상블 해봤으면 어떤 결과가 나올지 궁금하다. 베이스라인부터  ReduceLROnPlateau, ModelCheckPoint 같이 사용하자.
<br/>

- 성능 개선을 위한 더 적용해볼만한 방법
  - 1. 더 다양한 이미지 증강기법을 사용해볼것. 이번엔 단순한 회전, 반전 등 간단한 방법만 사용했지만 RGB값이나 컷팅을 통한 노이즈 넣기, 자르기, 명암 조절등 이미지 변형에는 많은 방법들이 존재한다. 데이터에 맞는 증강기법을 찾아보는 것도 좋은 실험이 될듯하다
  - 2. 더 세부적인 조정을 할 것. 초반 베이스라인 만들땐 EarlyStopoing하나만을, 그리고 나중에 전이학습부터 ReduceLROnPlateau, ModelCheckPoint 를 사용하였다. 이렇게 모듈에서 제공하는 간단한 기능들은 처음부터 적용하되 나중엔 SparseCategoricalCrossentropy같은 loss나 optimizer의 하이퍼 파라미터,  층에서 kernel_initializer 같은 웨이트 초기화 방법들도 다른 것들을 써보고 학습률도 직접 스케줄링 함수를 짜서 써보는 등, 다양한 하이퍼 파라미터를 조정하면서 어떤 변화가 있는지 실험해볼것.
  - 3. 데이터API를 더 세밀하게 사용해서 처리속도를 늘려볼 것. 그리고 하나의 파이프라인을 만들 것. 이번에는 완전 기초적인 데이터API를 사용했다. CPU와 GPU는 한정된 자원이므로 최대한 활용을 해야 더 많이 모델을 돌려서 피드백을 하면서 성능을 올릴 수 있다. 그러니 텐서플로의 가이드와 핸즈온 머신러닝, 각종 레퍼런스 등을 최대한 참고해서 적용시켜보자.
  - 4. Fine-tune 시 동결층을 좀더 줄여볼것. 전이학습시에는 크게 성능변화가 없었고 증강에 위에서 conv 1,2층 까지 열고 fine-tune시 0.2~3 정도 상승.  GPU자원이 부족해서 학습할 양이 늘어나면 중간에 할당을 다써버리기에 제대로 학습시키지 못했는데… 나중에 GPU 충분할때 시도해보자…



