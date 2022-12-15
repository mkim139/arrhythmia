# Arrhythmia Prediction

Heart Disease AI Datathon 2021에서 제공한 개인 심전도 (ECG) 데이터를 활용하여 부정맥 진단 모델 개발  
개인이 부정맥이 있을 가능성을 알려주며, model performance 99.7% (AUC) 달성

## 부정맥이란?

심장의 불규칙적인 리듬이나 비정상적인 심장박동을 의미 
진단 시, 심전도 그림을 보고 비정상적인 박동의 모양이 있는지 확인  
&#8594; 불규칙적 ‘모양’을 확인할 수 있는 모델을 만들 수 있을 것으로 보임 (CNN)  
&#8594; 시계열 분석으로 패턴이 들쑥날쑥한지 확인 할 수 있을 것으로 보임 (LSTM)  
대부분의 부정맥의 경우, 불규칙적 패턴이 명료하게 나타나나 (왼쪽 그림), 가끔 굉장히 짧은 비정상 패턴이 나타날 수 있음  
&#8594; 짧은 비정상 패턴에 더 집중하도록 Attention 메커니즘을 넣을 수도 있음  

## 데이터 구조

* 64bit encoding 데이터로 암호화 되어 있어 numerical sequence로 변환이 필요
* I, II, III aVF, aVL, aVR, V1~v6의 lead pulse sequence 데이터 제공

![image](https://user-images.githubusercontent.com/32697109/173257988-084fea3d-1f57-4a2d-9ee7-9d1a76dcbadf.png)

![image](https://user-images.githubusercontent.com/32697109/173257997-8e3e7ba7-d2ef-4391-8f53-9d1353eb3c08.png)

* 각 lead들은 다른 리드들의 선형적 조합으로 구성할 수 있어, 모두 사용해도 의미가 없을 것으로 생각, 몇 가지 lead만 사용하기로 결정  
* Sequence Length가 다른 경우가 있어 padding 등으로 맞추거나 오류 데이터의 경우 제거의 필요가 있음  
* 다른 질병 데이터와는 다르게 data imbalance 문제는 minor한 것으로 보임  
* 심전도 측정 시 사람들이 움직여서 생긴 기울기나 노이즈가 작은 anomaly를 Detect하는데 방해를 할 수 있어 개선이 필요해 보임  

![image](https://user-images.githubusercontent.com/32697109/173258049-42393a2e-db13-49b5-a974-d993723b9a25.png)

## 전처리 과정


![image](https://user-images.githubusercontent.com/32697109/173258084-d3859cff-c1fc-4607-a0b8-00cdaa34274e.png)
*기울어진 데이터*

![image](https://user-images.githubusercontent.com/32697109/173258095-e70e6746-d4d5-4160-b946-0f21a55c1336.png)
*Aligned 데이터*


1. 64bit encoding에서 Numerical Sequence로 변경
2. Neurokit package에서 제공하는 필터를 이용하여 움직임으로 생긴 듯한 기울어짐이나 노이즈 제거 (‘Neurokit Signal filter’)
3. I, II, III aVF, aVL, aVR sequence length를 체크하고 지정된 길이를 넘어가거나 부족할 시 자르거나 padding처리
4. 각 lead를 연결하여 하나의 긴 sequence로 만들고 전처리 마무리
(Normalization도 고려했으나 performance 개선에 유의미한 영향을 미치지 못함)

## 모델링

![image](https://user-images.githubusercontent.com/32697109/173258260-85ee287f-48a9-46aa-be3e-8899d1315a0e.png)


LSTM 모델  
LSTM을 활용하여 Sequence를 읽고, final hidden layer를 연속적인 fully connected layer들에 통과시켜 final binary (부정맥 여부) classification 시도 (Lead I,II,III 각각 모델을 만들어 퍼포먼스 확인)  
Maximum ~80% accuracy 달성  
Attention도 추가해 보았으나 개선 x  

Comment
* Sequence가 너무 길어 training time이 길어짐 (length = 1249)  
* 긴 sequence는 LSTM을 사용한다고 해도, vanishing gradient 또는 explosion을 초래 할 수 있음  
* 비효율적인 훈련 시간으로 다양한 구조를 시행해볼 시간이 부족  
&#8594; CNN구조를 사용하는 것으로 결정  

![image](https://user-images.githubusercontent.com/32697109/173258277-84cf4993-d3fd-41b8-bd87-00ce2d21b27e.png)

CNN Model  
* 1-dimensional CNN을 쌓아여 sequence pattern을 분석하도록 설계  
* 여러 lead sequence를 이어붙임 (LSTM과 다르게 시계열성에서 독립)  
* Multi-filter를 사용하여 다양한 너비의 view를 볼 수 있도록 구성 (kernel size 3, 5, 7 로 layer를 거친 후 concat)  
* Skip-connection 구조를 추가하여 deep한 모델을 만들고 vanishing gradient 문제를 개선 (얕은 모델에서 실험결과 ~.7% acc 개선, deep 한 경우 더 gap이 클 것으로 예상)  
* LSTM보다 더 빠른 훈련 속도를 보임 (~3s/it -> ~5it/s)  
* ACC: 97.4%, AUC: 99.7% 달성  


Performance 99.7% AUC (AUC curve)

![AUC curve](https://user-images.githubusercontent.com/32697109/173234422-f2352a0d-d97e-4bcc-bbf0-a97b40af2887.png)

Specificity Barplot  
![image](https://user-images.githubusercontent.com/32697109/207476181-8cb08659-a05d-4fe3-b73d-4c853009c446.png)


Class별 Probability Distribution  
![image](https://user-images.githubusercontent.com/32697109/207475135-a1201951-0a06-4907-8109-67d66bd2e3c5.png)  
* 0이 정상  
* 5,10 type이 가장 Prediction Probability가 낮게 나와 문제가 되고 있음  

Class 별 Count  
![image](https://user-images.githubusercontent.com/32697109/207475321-e10f8931-ba6c-4519-8514-608b93dd4c5f.png)  
* 5번의 경우 Sample Count가 현저히 낮아, Augmentation이나 이 Class에 더 높은 Loss가 필요해 보임  
* 10번의 경우 Sample Count는 높으나, 정상군과 많이 비슷한 형태를 가지고 있는것처럼 보임 (Type이름: Ectopic Wave)  

Type별 T-SNE 결과  
![image](https://user-images.githubusercontent.com/32697109/207475642-d2f4734d-a8d0-4ef4-8965-527f6cb496e2.png)
* 정상군도 여러 Type으로 묶이고 있음 (Batch Effect, Clinical Bias, Demographic Bias etc...)  
* Class 별로 Cluster가 잘되어 있는 경우도 있지만, 정상군의 Variance로 들어가는 Case들 또한 많음  
* Accuracy만 믿기에는 Sample 숫자가 많아 Bias된 인사이트를 가져올 수 있음  

정상군 Clustering에 대한 의문  
![image](https://user-images.githubusercontent.com/32697109/207988307-72dd9af2-3ad9-4f29-b687-bb97ef3e6bfe.png)  
* 정상군이 각기 다른 type으로 Cluster가 되는것을 T-SNE결과에서 확인함  
* 정상군을 Sample하여 확인해보면 Range가 각기 다른형태로 나타나는것을 볼 수 있음  
* Mean은 0에 몰려있음

![image](https://user-images.githubusercontent.com/32697109/207988665-ba935a44-208e-4e4b-b634-43d586605700.png)  
* 같은 정상군이라도 여러 Type으로 나뉘는것을 확인할 수 있음

## To-Be

![image](https://user-images.githubusercontent.com/32697109/173258375-c9194ee9-ea15-49e0-9657-57928000fdb3.png)
*Squeeze-and-Excite*

![image](https://user-images.githubusercontent.com/32697109/173258389-d5c18d10-1b89-474a-a80e-9fcb1fbef840.png)
*Ectopic Wave*

* MobileNet등에서 활용된 Squeeze-and-excite과 같은 attention approach를 활용하여 불필요한 feature에 attend를 덜하는 방식을 고려해 봤을 수 있을 것 같음  
&#8594; 작은 anomaly pattern의 경우, 다른 normal pattern에 	overridden 되어 충분한 weight을 얻지 못했을 수 있음  
&#8594; 실제로 Ectopic Atrial Rhythm의 error rate은 54%정도였고, 일반인의 눈으로는 부정맥임을 판단하기 어려울 정도로 작은 anomal 	pattern임 (작은 Kernel size를 활용하여 개선이 되었으나, 여전히 가장 높은 error rate을 보임)  





