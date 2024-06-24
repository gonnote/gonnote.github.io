<<<<<<< HEAD
---
title:  "[AUTOSAR] DCM"
excerpt: "리프로그래밍 시에만 부트로 점프하지 않는다. DCM의 서비스 전환 과정도 부트로더와 관련 있다."

categories:
  - Autosar
tags:
  - [Autosar, UDS ON CAN, Diagnosis]

toc: true
toc_sticky: true
 
date: 2024-03-05
last_modified_at: 2024-03-05
---


# DCM

> 10 서비스에 대한 부트로 점프 요청을 처리하는 과정은 DCM에서 이루어짐

### ■ 부트로 점프하는 Process와 관련된 DCM Config Parameter
### &nbsp;&nbsp;□ DcmDspSessionForBoot
- 진단 세션을 부트로 전환해야 하는 것으로 설정한다.
- OEM와 공급 업체를 위한 두개의 분리된 부트로더를 식별할 수 있고 사용은 ECU에 대한 BSW의 설계에 달려 있다. 
### &nbsp;&nbsp;□ DcmSendRespPendOnTransToBoot
- 부트로더로 점프 작업이 수행되기 전 NRC_RCRRP 0x78 응답을 해야 하는 경우 이 Parameter를 사용하여 활성화할 수 있다. <br>
  ( P2*Server 타임아웃이 발생하지 않도록 하기 위해 수행된다.)<br>
  &#8251;RCRRP : **R**equest **C**orrectly **R**eceived, but **R**esponse is **P**ending<br>
    ( ECU가 요청을 정확하게 받았지만 응답을 위해 시간이 필요할 때 진단기에 알릴 때 사용된다. )
### &nbsp;&nbsp;□ ModeDeclarationGroupPrototype DcmEcuReset
- 작업이 요청되면 BsWm은 먼저 JUMPTOBOOTLOADER 또는 JUMPTOSYSSUPLIERBOOTLOADER로 전환되고, 모든 검사가 수행되고 ECU가 리셋 준비가 완료되면 EXECUTE로 전환된다.

### ■ 부트 점프 절차
### &nbsp;&nbsp;1.BswM 트리거 위한 DcmEcuReset의 모드 전환
> BswM을 트리거한다?<br>
> : BswM의 동작을 시작하거나 특정 모드로 전환하도록 하는 것을 의미<br>
>&nbsp;특정 조건이 충족될 때 BswM의 동작을 시작하거나 변경하도록 하는 신호나 이벤트<br>

> &#8251; 트리거의 Example
> 1. ECU 상태 변경: ECU의 상태가 변경되면 BswM은 해당 트리거를 수신하고 모드 전환 수행
> 2. DTC 상태 변경: 진단 문제 코드(DTC)의 상태가 변경되면 이 정보는 BswM에 전달될 수 있으며, 이는 모드 전환을 트리거 가능 
> 3. 직접적인 요청: 특정 BSW 모듈(예: DcmEcuReset)에서 직접 BswM 모드 전환을 요청 가능 
- DcmDspSessionForBoot의 값에 따라 ModeDeclarationGroupPrototype DcmEcuReset의 모드가 JUMPTOBOOTLOADER 또는 JUMPTOSYSSUPPLIERBOOTLOADER로
  변경된다.<br>
만약 DcmDspSessionForBoot가 DCM_OEM_BOOT 또는 DCM_OEM_BOOT_RESPAPP로 설정되면 DcmEcuReset은 JUMPTOBOOTLOADER로 설정된다.
&rightarrow; OEM 부트로더로 수행<br>
만약 DcmDspSessionForBoot가 DCM_SYS_BOOT 또는 DCM_SYS_BOOT_RESPAPP로 설정되면 DcmEcuReset은 JUMPTOSYSSUPPLIERBOOTLOADER로 설정된다.
&rightarrow; 공급업체 부트로더로 수행<br>
만약 이 단계가 실패하면 부트로더 동작의 점프를 종료하고 NRC 22를 보낸다.
### &nbsp;&nbsp;2. 응답 보류 중인 부정 응답 코드를 전송
- DcmSendRespPendOnTransToBoot가 TRUE로 설정되어 있는 경우, 응답 보류 중인 부정 응답(NRC_RCRRP_0x78)이 전송된다.<br>
  이 단계가 실패하면 부트로더의 점프가 중단되고 다음 요청을 위해 진단 버퍼가 unlock 된다.<br>
  세션 전환 요청이 suppressPosRspMsgIndicationBit = 1로 설정되고 DcmSendRespPendOnTransToBoot가 TRUE로 설정되면 최종 긍정 응답이 전송된다.<br>
  응답 보류 중인 부정 응답(NRC_RCRRP_0x78)이 전송되었을 때는 긍정 응답의 Suppress가 무시된다.<br>
  &#8251; suppressPosRspMsgIndicationBit : 1(TRUE) &rightarrow; 긍정 응답 존재, 0(FALSE) &rightarrow; 긍정 응답 없음
### &nbsp;&nbsp;3. 최종 응답 전송
- DcmDspSessionForBoot이 DCM_SYS_BOOT_RESPAPP 또는 DCM_OEM_BOOT_RESPAPP으로 설정된 경우에만 적용된다.<br>
  이것은 ECU가 부트로더로 이동을 위해 RESET 전에 DCM이 보낼 최종 긍정 응답을 처리하는 단계입니다
### &nbsp;&nbsp;4. Dcm_SetProgConditions() 호출
- 어플리케이션이 부트로더로 이동하기 전에 관련 정보를 저장할 수 있게 한다.  
  E_OK를 반환하면 다음 단계로 넘어가고 E_NOT_OK를 반환하면 NRC 0x22 부정 응답으로 종료된다.
### &nbsp;&nbsp;5. DcmEcuReset이 EXECUTE 상태로 MODE SWTICH
-  모든 검사가 완료되고 필요한 응답들이 전송되면 DcmEcuReset의 모드가 EXECUTE로 변경된다. <br>
   어플리케이션에서 ECU Reset을 수행 가능한 상태다. 
### &nbsp;&nbsp;6. ECU Reset 수행
- 리셋이 수행되는 부분이며, 첫번째 명령어를 강제로 실행할 때 트리거된다. <br>
  또는, 와치독 리셋으로 트리거될 수 있다. 
  










```python

```


=======
---
title:  "[AUTOSAR] DCM"
excerpt: "리프로그래밍 시에만 부트로 점프하지 않는다. DCM의 서비스 전환 과정도 부트로더와 관련 있다."

categories:
  - Autosar
tags:
  - [Autosar, UDS ON CAN, Diagnosis]

toc: true
toc_sticky: true
 
date: 2024-03-05
last_modified_at: 2024-03-05
---


# DCM

> 10 서비스에 대한 부트로 점프 요청을 처리하는 과정은 DCM에서 이루어짐

### ■ 부트로 점프하는 Process와 관련된 DCM Config Parameter
### &nbsp;&nbsp;□ DcmDspSessionForBoot
- 진단 세션을 부트로 전환해야 하는 것으로 설정한다.
- OEM와 공급 업체를 위한 두개의 분리된 부트로더를 식별할 수 있고 사용은 ECU에 대한 BSW의 설계에 달려 있다. 
### &nbsp;&nbsp;□ DcmSendRespPendOnTransToBoot
- 부트로더로 점프 작업이 수행되기 전 NRC_RCRRP 0x78 응답을 해야 하는 경우 이 Parameter를 사용하여 활성화할 수 있다. <br>
  ( P2*Server 타임아웃이 발생하지 않도록 하기 위해 수행된다.)<br>
  &#8251;RCRRP : **R**equest **C**orrectly **R**eceived, but **R**esponse is **P**ending<br>
    ( ECU가 요청을 정확하게 받았지만 응답을 위해 시간이 필요할 때 진단기에 알릴 때 사용된다. )
### &nbsp;&nbsp;□ ModeDeclarationGroupPrototype DcmEcuReset
- 작업이 요청되면 BsWm은 먼저 JUMPTOBOOTLOADER 또는 JUMPTOSYSSUPLIERBOOTLOADER로 전환되고, 모든 검사가 수행되고 ECU가 리셋 준비가 완료되면 EXECUTE로 전환된다.

### ■ 부트 점프 절차
### &nbsp;&nbsp;1.BswM 트리거 위한 DcmEcuReset의 모드 전환
> BswM을 트리거한다?<br>
> : BswM의 동작을 시작하거나 특정 모드로 전환하도록 하는 것을 의미<br>
>&nbsp;특정 조건이 충족될 때 BswM의 동작을 시작하거나 변경하도록 하는 신호나 이벤트<br>

> &#8251; 트리거의 Example
> 1. ECU 상태 변경: ECU의 상태가 변경되면 BswM은 해당 트리거를 수신하고 모드 전환 수행
> 2. DTC 상태 변경: 진단 문제 코드(DTC)의 상태가 변경되면 이 정보는 BswM에 전달될 수 있으며, 이는 모드 전환을 트리거 가능 
> 3. 직접적인 요청: 특정 BSW 모듈(예: DcmEcuReset)에서 직접 BswM 모드 전환을 요청 가능 
- DcmDspSessionForBoot의 값에 따라 ModeDeclarationGroupPrototype DcmEcuReset의 모드가 JUMPTOBOOTLOADER 또는 JUMPTOSYSSUPPLIERBOOTLOADER로
  변경된다.<br>
만약 DcmDspSessionForBoot가 DCM_OEM_BOOT 또는 DCM_OEM_BOOT_RESPAPP로 설정되면 DcmEcuReset은 JUMPTOBOOTLOADER로 설정된다.
&rightarrow; OEM 부트로더로 수행<br>
만약 DcmDspSessionForBoot가 DCM_SYS_BOOT 또는 DCM_SYS_BOOT_RESPAPP로 설정되면 DcmEcuReset은 JUMPTOSYSSUPPLIERBOOTLOADER로 설정된다.
&rightarrow; 공급업체 부트로더로 수행<br>
만약 이 단계가 실패하면 부트로더 동작의 점프를 종료하고 NRC 22를 보낸다.
### &nbsp;&nbsp;2. 응답 보류 중인 부정 응답 코드를 전송
- DcmSendRespPendOnTransToBoot가 TRUE로 설정되어 있는 경우, 응답 보류 중인 부정 응답(NRC_RCRRP_0x78)이 전송된다.<br>
  이 단계가 실패하면 부트로더의 점프가 중단되고 다음 요청을 위해 진단 버퍼가 unlock 된다.<br>
  세션 전환 요청이 suppressPosRspMsgIndicationBit = 1로 설정되고 DcmSendRespPendOnTransToBoot가 TRUE로 설정되면 최종 긍정 응답이 전송된다.<br>
  응답 보류 중인 부정 응답(NRC_RCRRP_0x78)이 전송되었을 때는 긍정 응답의 Suppress가 무시된다.<br>
  &#8251; suppressPosRspMsgIndicationBit : 1(TRUE) &rightarrow; 긍정 응답 존재, 0(FALSE) &rightarrow; 긍정 응답 없음
### &nbsp;&nbsp;3. 최종 응답 전송
- DcmDspSessionForBoot이 DCM_SYS_BOOT_RESPAPP 또는 DCM_OEM_BOOT_RESPAPP으로 설정된 경우에만 적용된다.<br>
  이것은 ECU가 부트로더로 이동을 위해 RESET 전에 DCM이 보낼 최종 긍정 응답을 처리하는 단계입니다
### &nbsp;&nbsp;4. Dcm_SetProgConditions() 호출
- 어플리케이션이 부트로더로 이동하기 전에 관련 정보를 저장할 수 있게 한다.  
  E_OK를 반환하면 다음 단계로 넘어가고 E_NOT_OK를 반환하면 NRC 0x22 부정 응답으로 종료된다.
### &nbsp;&nbsp;5. DcmEcuReset이 EXECUTE 상태로 MODE SWTICH
-  모든 검사가 완료되고 필요한 응답들이 전송되면 DcmEcuReset의 모드가 EXECUTE로 변경된다. <br>
   어플리케이션에서 ECU Reset을 수행 가능한 상태다. 
### &nbsp;&nbsp;6. ECU Reset 수행
- 리셋이 수행되는 부분이며, 첫번째 명령어를 강제로 실행할 때 트리거된다. <br>
  또는, 와치독 리셋으로 트리거될 수 있다. 
  










```python

```


>>>>>>> c27dbf2309c0cd483a472cbab4662f3073e91d39
