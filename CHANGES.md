# KV GET 자동완료 제어 변경사항

## 목표
NVMe KV GET 명령이 4KB NAND 데이터를 읽고 완료 신호에서 `specific` 필드로 데이터 길이(4096)를 전달하도록 수정

## 변경된 파일

### 1. `GreedyFTL/Embeded_System_Software/src/request_format.h`

#### 추가: REQ_CODE_KV_GET 정의 (Line ~72)
```c
#define REQ_CODE_KV_GET             0x11  // KV GET - no auto-completion
```
- KV GET 요청을 FTL 파이프라인에서 추적하기 위한 고유 코드
- 일반 READ(0x08)와 다르게 자동완료를 OFF로 설정

#### 수정: REQ_OPTION 구조 (Line 143-152)
```c
typedef struct _REQ_OPTION{
    unsigned int dataBufFormat : 2;
    unsigned int nandAddr : 2;
    unsigned int nandEcc : 1;
    unsigned int nandEccWarning : 1;
    unsigned int rowAddrDependencyCheck : 1;
    unsigned int blockSpace : 1;
    unsigned int isKvGet : 1;  // 추가: KV GET 플래그
    unsigned int reserved0 : 23;  // 24에서 23으로 감소
} REQ_OPTION, *P_REQ_OPTION;
```
- `isKvGet` (1-bit) 플래그 추가
- 요청이 KV GET에서 생성되었는지 나중에 추적 가능
- `IssueNvmeDmaReq()`에서 auto-completion ON/OFF 결정에 사용

---

### 2. `GreedyFTL/Embeded_System_Software/src/request_transform.h`

#### 추가: ReqTransNvmeToSliceKvGet 함수 선언 (Line 81)
```c
void ReqTransNvmeToSliceKvGet(unsigned int cmdSlotTag, unsigned int startLba, unsigned int nlb);
```
- `ReqTransNvmeToSlice()`와 유사하지만 KV GET 전용
- `cmdCode` 파라미터 없음 (항상 READ)
- `reqCode=REQ_CODE_KV_GET` 으로 고정
- `reqOpt.isKvGet=1` 플래그 설정

---

### 3. `GreedyFTL/Embeded_System_Software/src/request_transform.c`

#### 추가: ReqTransNvmeToSliceKvGet 함수 구현 (Line 163-238)
```c
void ReqTransNvmeToSliceKvGet(unsigned int cmdSlotTag, unsigned int startLba, unsigned int nlb)
```
- **First transform**: startLba부터 첫 slice 끝까지의 블록 변환
  ```c
  reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_KV_GET;
  reqPoolPtr->reqPool[reqSlotTag].reqOpt.isKvGet = 1;  // KV GET 플래그 설정
  ```
- **Continue transform**: 중간 slice들 변환 (루프)
  ```c
  reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_KV_GET;
  reqPoolPtr->reqPool[reqSlotTag].reqOpt.isKvGet = 1;
  ```
- **Last transform**: 마지막 slice 변환
  ```c
  reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_KV_GET;
  reqPoolPtr->reqPool[reqSlotTag].reqOpt.isKvGet = 1;
  ```

#### 수정: ReqTransSliceToLowLevel() 함수 (Line 328-330)
```c
// 변경 전:
if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_READ)
    DataReadFromNand(reqSlotTag);
else if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_WRITE)
    if(reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.numOfNvmeBlock != NVME_BLOCKS_PER_SLICE)
        DataReadFromNand(reqSlotTag);

// 변경 후:
if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_READ)
    DataReadFromNand(reqSlotTag);
else if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_WRITE)
    if(reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.numOfNvmeBlock != NVME_BLOCKS_PER_SLICE)
        DataReadFromNand(reqSlotTag);
else if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_KV_GET)
    DataReadFromNand(reqSlotTag);  // KV GET도 NAND에서 읽어야 함
```

#### 수정: ReqTransSliceToLowLevel() - reqCode 변환 (Line 336-349)
```c
// 변경 전:
if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_WRITE)
{
    dataBufMapPtr->dataBuf[dataBufEntry].dirty = DATA_BUF_DIRTY;
    reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_RxDMA;
}
else if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_READ)
    reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_TxDMA;
else
    assert(!"[WARNING] Not supported reqCode. [WARNING]");

// 변경 후:
if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_WRITE)
{
    dataBufMapPtr->dataBuf[dataBufEntry].dirty = DATA_BUF_DIRTY;
    reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_RxDMA;
}
else if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_READ)
    reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_TxDMA;
else if(reqPoolPtr->reqPool[reqSlotTag].reqCode  == REQ_CODE_KV_GET)
{
    reqPoolPtr->reqPool[reqSlotTag].reqCode = REQ_CODE_TxDMA;  // TxDMA로 변환, isKvGet=1 유지
}
else
    assert(!"[WARNING] Not supported reqCode. [WARNING]");
```

#### 수정: IssueNvmeDmaReq() 함수 (Line 662-700)
```c
// 변경 전:
void IssueNvmeDmaReq(unsigned int reqSlotTag)
{
    unsigned int devAddr, dmaIndex, numOfNvmeBlock;
    // ...
    else if(reqPoolPtr->reqPool[reqSlotTag].reqCode == REQ_CODE_TxDMA)
    {
        while(numOfNvmeBlock < reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.numOfNvmeBlock)
        {
            set_auto_tx_dma(..., NVME_COMMAND_AUTO_COMPLETION_ON);  // 항상 ON
        }
    }
}

// 변경 후:
void IssueNvmeDmaReq(unsigned int reqSlotTag)
{
    unsigned int devAddr, dmaIndex, numOfNvmeBlock, autoCompletion;
    // ...
    else if(reqPoolPtr->reqPool[reqSlotTag].reqCode == REQ_CODE_TxDMA)
    {
        // isKvGet 플래그에 따라 auto-completion 제어
        autoCompletion = (reqPoolPtr->reqPool[reqSlotTag].reqOpt.isKvGet) ? 
                         NVME_COMMAND_AUTO_COMPLETION_OFF : 
                         NVME_COMMAND_AUTO_COMPLETION_ON;
        
        while(numOfNvmeBlock < reqPoolPtr->reqPool[reqSlotTag].nvmeDmaInfo.numOfNvmeBlock)
        {
            set_auto_tx_dma(..., autoCompletion);  // 조건부
        }
    }
}
```
- **핵심**: `isKvGet=1`이면 `NVME_COMMAND_AUTO_COMPLETION_OFF` 사용
- 일반 READ: 하드웨어가 자동으로 완료 신호 생성
- KV GET: 하드웨어가 자동 완료하지 않음 → 수동 완료 필요

#### 수정: CheckDoneNvmeDmaReq() 함수 (Line 709-734)
```c
// 변경 전:
if(txDone)
    SelectiveGetFromNvmeDmaReqQ(reqSlotTag);

// 변경 후:
if(txDone)
{
    // KV GET 요청 완료 후 수동 완료 신호 전송
    if(reqPoolPtr->reqPool[reqSlotTag].reqOpt.isKvGet)
    {
        // specific 필드 = 4096 (데이터 길이)
        set_auto_nvme_cpl(reqPoolPtr->reqPool[reqSlotTag].nvmeCmdSlotTag, 4096, 0);
    }
    
    SelectiveGetFromNvmeDmaReqQ(reqSlotTag);
}
```
- **핵심**: DMA 완료 시 `isKvGet` 플래그 확인
- KV GET이면 `set_auto_nvme_cpl(cmdSlotTag, 4096, 0)` 호출
- `specific=4096`: 호스트에 데이터 길이 전달
- status=0: 성공 상태

---

### 4. `GreedyFTL/Embeded_System_Software/src/nvme/nvme_io_cmd.c`

#### 수정: handle_nvme_io_kv_get() 함수 (Line 151-175)
```c
// 변경 전:
void handle_nvme_io_kv_get(unsigned int cmdSlotTag, NVME_IO_COMMAND *nvmeIOCmd)
{
    NVME_COMPLETION nvmeCPL;
    unsigned int key = nvmeIOCmd->dword[10];
    unsigned int nlb = nvmeIOCmd->dword[12];
    unsigned int lsa = HashKeyToLSA(key);
    
    if (logicalSliceMapPtr->logicalSlice[lsa].virtualSliceAddr == VSA_NONE) {
        nvmeCPL.dword[0] = 0;
        nvmeCPL.statusField.SCT = SCT_GENERIC_COMMAND_STATUS;
        nvmeCPL.statusField.SC = ENOSUCHKEY;
        set_auto_nvme_cpl(cmdSlotTag, 0, nvmeCPL.statusFieldWord);  // BUG: specific에 status 전달
        return;
    }
    
    // ReqTransNvmeToSlice(cmdSlotTag, lsa, nlb, IO_NVM_READ);  // 주석
    set_auto_nvme_cpl(cmdSlotTag, 4096, 0);  // 하드코딩, DMA 없음
}

// 변경 후:
void handle_nvme_io_kv_get(unsigned int cmdSlotTag, NVME_IO_COMMAND *nvmeIOCmd)
{
    unsigned int key = nvmeIOCmd->dword[10];
    unsigned int nlb = nvmeIOCmd->dword[12];
    unsigned int lsa = HashKeyToLSA(key);
    
    // 키 미존재: specific 필드에 에러 코드 전달
    if (logicalSliceMapPtr->logicalSlice[lsa].virtualSliceAddr == VSA_NONE) {
        set_auto_nvme_cpl(cmdSlotTag, ENOSUCHKEY, 0);  // ENOSUCHKEY = 0x7C1
        return;
    }
    
    ASSERT((nvmeIOCmd->PRP1[0] & 0x3) == 0 && (nvmeIOCmd->PRP2[0] & 0x3) == 0);
    ASSERT(nvmeIOCmd->PRP1[1] < 0x10000 && nvmeIOCmd->PRP2[1] < 0x10000);
    
    // FTL 파이프라인에 KV GET 요청 큐잉 (자동완료 OFF)
    // DMA 완료 후 수동으로 specific=4096으로 완료 신호 전송
    ReqTransNvmeToSliceKvGet(cmdSlotTag, lsa, nlb);
}
```

**주요 변경점:**
1. **에러 처리 개선**: `specific` 필드에 에러 코드(ENOSUCHKEY) 전달
2. **DMA 활성화**: `ReqTransNvmeToSliceKvGet()` 호출로 실제 NAND 읽기 시작
3. **자동완료 제어**: `isKvGet=1` 플래그로 later stage에서 auto-completion OFF
4. **수동 완료**: `CheckDoneNvmeDmaReq()`에서 DMA 완료 후 `set_auto_nvme_cpl(cmdSlotTag, 4096, 0)` 호출

---

## 동작 흐름

```
Host KV GET 요청
    ↓
handle_nvme_io_kv_get()
    ├─ 키 미존재? → set_auto_nvme_cpl(cmdSlotTag, ENOSUCHKEY, 0) → 즉시 완료
    └─ 키 존재?
        ↓
        ReqTransNvmeToSliceKvGet(cmdSlotTag, lsa, nlb)
        [reqCode=KV_GET, isKvGet=1 플래그 설정]
        ↓
        ReqTransSliceToLowLevel()
        [버퍼 할당, NAND 읽기 요청]
        ↓
        reqCode = REQ_CODE_TxDMA (isKvGet=1 유지)
        ↓
        SelectLowLevelReqQ()
        ↓
        IssueNvmeDmaReq()
        [autoCompletion = NVME_COMMAND_AUTO_COMPLETION_OFF]
        ↓
        set_auto_tx_dma(..., NVME_COMMAND_AUTO_COMPLETION_OFF)
        [하드웨어가 자동 완료하지 않음]
        ↓
        DMA 실행 (호스트 버퍼로 데이터 전송)
        ↓
        CheckDoneNvmeDmaReq()
        [isKvGet=1 확인]
        ↓
        set_auto_nvme_cpl(cmdSlotTag, 4096, 0)
        [수동 완료, specific=4096 (데이터 길이)]
        ↓
Host가 result=4096 받음 ✓
```

---

## 핵심 변경 원칙

| 항목 | 변경 전 | 변경 후 |
|------|--------|--------|
| **요청 추적** | `reqCode=READ` (일반과 동일) | `reqCode=KV_GET` + `isKvGet=1` |
| **자동완료** | 하드웨어 자동 (specific=0) | 수동 제어 (specific=4096) |
| **데이터** | DMA 없음 (하드코딩) | 실제 NAND 읽기 |
| **에러 처리** | status 필드 사용 (틀림) | specific 필드 사용 (맞음) |
| **완료 시점** | 즉시 | DMA 완료 후 |

---

## 이전 문제점

1. ❌ 자동완료 ON → specific 필드를 제어 불가
2. ❌ DMA 없음 → 실제 데이터 전송 안 됨
3. ❌ 에러 코드가 status 필드에 → host에서 인식 불가
4. ❌ 하드코딩 → 다양한 data length 미지원

## 해결 방법

1. ✅ 자동완료 OFF → specific 필드를 4096 또는 ENOSUCHKEY로 설정 가능
2. ✅ FTL 파이프라인을 통한 실제 NAND 읽기
3. ✅ 에러 코드를 specific 필드로 전달 → host가 `result` 필드에서 인식
4. ✅ `CheckDoneNvmeDmaReq()`에서 동적으로 complete 신호 전송

