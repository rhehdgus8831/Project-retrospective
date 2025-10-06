## 🧯 트러블슈팅

### **문제: 모델의 높은 검증 정확도와 실제 테스트 성능의 불일치**

#### **문제 상황**

1.  데이터 증강, 모델 경량화, 3D 좌표 변환 등 다양한 기술적 튜닝을 통해 Transformer 모델의 **최고 검증 정확도를 84%까지 달성**했습니다.
  
   <img width="671" height="194" alt="스크린샷 2025-10-07 000913" src="https://github.com/user-attachments/assets/95e7865d-e12d-4bf8-973d-c695ab3d3631" />

2.  특히 '기쁨', '인사' 등 일부 테스트 영상에서는 정답을 맞히는 등 성능 개선의 가능성을 확인했습니다.
3.  하지만 학습 데이터에 없던 대부분의 새로운 테스트 영상을 입력했을 때, 모델은 여전히 특정 단어('미국', '인사' 등)로만 예측하는 **일반화(Generalization) 실패 문제**가 지속적으로 발생했습니다.
   
```
============================================================
✨ 학습 완료!
📁 모델 저장 위치: models
📊 최고 검증 정확도: 84.00%
 PC   sign_language_videos   transformer ≢  ?125 ~6  +192 -1                                                                                                                                       in pwsh at 00:08:44                                                                                                                                                                                                                                    
 sign_language_videos  # 다른 영상(예: 'hello.mp4')을 테스트하고 싶을 경우
 sign_language_videos  python test_video.py --video "test_video/math_test.mp4"
🔥 Device: cpu
✅ 모델 로드 완료: models\transformer_model.pth
   - 학습된 단어: ['기쁨' '미국' '수학' '안녕' '월세' '인사' '일요일']

[1/2] 'math_test.mp4' 영상에서 3D 좌표를 추출합니다...
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
W0000 00:00:1759763865.093263   26444 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.114138    1448 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.116853   24760 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.117184   10760 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.118069   26444 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.123405   26444 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.129738   26444 landmark_projection_calculator.cc:186] Using NORM_RECT without IMAGE_DIMENSIONS is only supported for the square ROI. Provide IMAGE_DIMENSIONS or use PROJECTION_MATRIX.
W0000 00:00:1759763865.134494   24760 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763865.134641   10760 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
[2/2] 모델 예측을 수행합니다...

==================================================
🎯 수어 인식 결과 (유사도 순위)
==================================================
1. 미국         | ███████████  38.0%
2. 인사         | █████████  32.9%
3. 안녕         | ███████  23.9%
4. 수학         |    2.9%
5. 월세         |    1.3%
==================================================
 sign_language_videos  python test_video.py --video "test_video/delight_test.mp4"
🔥 Device: cpu
✅ 모델 로드 완료: models\transformer_model.pth
   - 학습된 단어: ['기쁨' '미국' '수학' '안녕' '월세' '인사' '일요일']

[1/2] 'delight_test.mp4' 영상에서 3D 좌표를 추출합니다...
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
W0000 00:00:1759763888.657833   25760 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.679738    9500 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.682224    8952 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.682756   25760 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.683350    9500 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.688231    9500 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.694866   24572 landmark_projection_calculator.cc:186] Using NORM_RECT without IMAGE_DIMENSIONS is only supported for the square ROI. Provide IMAGE_DIMENSIONS or use PROJECTION_MATRIX.
W0000 00:00:1759763888.698661    8952 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763888.699495    9764 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
[2/2] 모델 예측을 수행합니다...

==================================================
🎯 수어 인식 결과 (유사도 순위)
==================================================
1. 기쁨         | █████████████  45.3%
2. 일요일        | █████████  30.3%
3. 미국         | ███  10.1%
4. 안녕         | █   4.0%
5. 인사         | █   3.8%
==================================================
 sign_language_videos  python test_video.py --video "test_video/amrerica_test.mp4"
🔥 Device: cpu
✅ 모델 로드 완료: models\transformer_model.pth
   - 학습된 단어: ['기쁨' '미국' '수학' '안녕' '월세' '인사' '일요일']
오류: 영상 파일 'test_video\amrerica_test.mp4'를 찾을 수 없습니다.
 sign_language_videos  python test_video.py --video "test_video/america_test.mp4"
🔥 Device: cpu
✅ 모델 로드 완료: models\transformer_model.pth
   - 학습된 단어: ['기쁨' '미국' '수학' '안녕' '월세' '인사' '일요일']

[1/2] 'america_test.mp4' 영상에서 3D 좌표를 추출합니다...
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
W0000 00:00:1759763916.853556   24912 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.877221   26436 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.879822   25096 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.880119   13672 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.880957   23756 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.886902   24912 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.893915   23252 landmark_projection_calculator.cc:186] Using NORM_RECT without IMAGE_DIMENSIONS is only supported for the square ROI. Provide IMAGE_DIMENSIONS or use PROJECTION_MATRIX.
W0000 00:00:1759763916.895922   10320 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763916.897185   25096 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
[2/2] 모델 예측을 수행합니다...

==================================================
🎯 수어 인식 결과 (유사도 순위)
==================================================
1. 인사         | ████████████  41.8%
2. 미국         | ███████  25.0%
3. 안녕         | ███████  23.7%
4. 수학         | ██   7.2%
5. 기쁨         |    1.2%
==================================================
 sign_language_videos  python test_video.py --video "test_video/greeting_test.mp4"
🔥 Device: cpu
✅ 모델 로드 완료: models\transformer_model.pth
   - 학습된 단어: ['기쁨' '미국' '수학' '안녕' '월세' '인사' '일요일']

[1/2] 'greeting_test.mp4' 영상에서 3D 좌표를 추출합니다...
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
W0000 00:00:1759763936.186018    5700 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.204731   18064 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.207631    5700 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.207765   16308 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.207971   25496 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.213334    9984 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.220607   20156 landmark_projection_calculator.cc:186] Using NORM_RECT without IMAGE_DIMENSIONS is only supported for the square ROI. Provide IMAGE_DIMENSIONS or use PROJECTION_MATRIX.
W0000 00:00:1759763936.223157   22756 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763936.224637    5700 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
[2/2] 모델 예측을 수행합니다...

==================================================
🎯 수어 인식 결과 (유사도 순위)
==================================================
1. 인사         | ██████████  35.9%
2. 미국         | ██████████  33.9%
3. 안녕         | ██████  23.1%
4. 수학         | █   4.5%
5. 기쁨         |    1.7%
==================================================
 sign_language_videos  python test_video.py --video "test_video/test.mp4"
🔥 Device: cpu
✅ 모델 로드 완료: models\transformer_model.pth
   - 학습된 단어: ['기쁨' '미국' '수학' '안녕' '월세' '인사' '일요일']

[1/2] 'test.mp4' 영상에서 3D 좌표를 추출합니다...
INFO: Created TensorFlow Lite XNNPACK delegate for CPU.
WARNING: All log messages before absl::InitializeLog() is called are written to STDERR
W0000 00:00:1759763949.275782   10716 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.294588   16212 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.297165     256 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.297471   11716 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.298177   16460 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.303590   16460 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.310703    1536 landmark_projection_calculator.cc:186] Using NORM_RECT without IMAGE_DIMENSIONS is only supported for the square ROI. Provide IMAGE_DIMENSIONS or use PROJECTION_MATRIX.
W0000 00:00:1759763949.313151   11716 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
W0000 00:00:1759763949.313781     256 inference_feedback_manager.cc:114] Feedback manager requires a model with a single signature inference. Disabling support for feedback tensors.
[2/2] 모델 예측을 수행합니다...

==================================================
🎯 수어 인식 결과 (유사도 순위)
==================================================
1. 미국         | ███████████████████████  78.8%
2. 기쁨         | ███  11.9%
3. 인사         |    3.0%
4. 월세         |    2.4%
5. 안녕         |    2.0%
==================================================

```



#### **원인 분석**

-   가장 큰 원인은 **데이터의 절대적인 양과 다양성 부족**이었습니다. 모델이 수어의 핵심 동작 패턴을 학습한 것이 아니라, 적은 양의 학습 데이터에만 존재하는 특정인의 손 모양, 촬영 각도, 속도 등 **피상적인 특징에 과적합(Overfitting)**된 상태였습니다.
-   결과적으로 84%라는 검증 정확도는, 학습 데이터와 유사한 패턴을 가진 검증 데이터셋에만 국한된 **'가짜 정확도'**였습니다. 모델이 진짜 실력을 갖췄다기보다는, 익숙한 '연습문제'만 잘 푸는 상태였던 것입니다.
-   이 과정에서 2D 좌표를 3D로 전환하며 `ValueError: operands could not be broadcast together with shapes (57,3) (2,)` 오류를 겪기도 했습니다. 이는 3D 좌표 정규화 과정에서 기준점(`origin`)을 2D 벡터로 잘못 설정한 코드 문제였으며, 3D 벡터 `np.array([0.0, 0.0, 0.0])`로 수정하여 해결했습니다. 이로써 데이터 처리 파이프라인의 문제는 모두 해결되었음을 확인했습니다.

#### **해결 방법**

-   **접근 방식 전환**: 더 이상의 코드 수정이나 모델 구조 변경만으로는 이 문제를 해결할 수 없다고 판단했습니다. '모델 중심(Model-centric)' 접근에서 **'데이터 중심(Data-centric)' 접근으로 전략을 전환**했습니다.
-   **체계적인 오류 분석**: 단순히 테스트를 반복하는 대신, 어떤 단어를 어떤 단어로 잘못 예측하는지 표로 정리하는 **'오답노트(Confusion Matrix)'**를 작성하여 모델의 취약점을 명확히 분석하기로 했습니다.
-   **목표 지향적 데이터 수집**: 분석된 취약점을 바탕으로, 모델이 특히 **자주 혼동하는 단어들을 위주**로, **다양한 사람과 환경**에서 촬영한 데이터를 추가 수집하여 데이터셋의 양과 질을 높이는 것을 최종 해결책으로 결정했습니다.

### **느낀 점**

> 이번 트러블슈팅을 통해, 딥러닝 프로젝트에서 검증 정확도라는 숫자가 얼마나 허무할 수 있는지 뼈저리게 느꼈습니다. 84%라는 숫자를 봤을 때 정말 모든 문제가 해결된 줄 알았지만, 실제 테스트 결과 앞에서 무너지는 것을 보며 큰 좌절감을 느꼈습니다.
>
> 하지만 이 과정을 통해 코드를 수정하고 파라미터를 조정하는 것만이 AI 개발의 전부가 아니라는 값진 교훈을 얻었습니다. 오히려 문제의 근본 원인이 '코드'가 아닌 '데이터'에 있다는 것을 스스로 증명해냈고, 이제는 '어떻게' 만들지가 아닌 **'무엇을' 학습시킬지**에 대해 고민해야 하는 더 중요한 단계로 나아갈 수 있게 되었습니다.
>
> 오늘 마주한 이 '일반화의 벽'은 실패가 아니라, 우리 프로젝트가 나아가야 할 가장 명확한 방향을 알려준 이정표였다고 생각합니다.
