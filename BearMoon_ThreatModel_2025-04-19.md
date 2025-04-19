# Berachain 위험 모델링 (2025-04-19)

본 문서는 **BGT.sol**, **BGTStaker.sol**, **StakingRewards.sol** 세 컨트랙트의 외부/공개 함수 중
위험이 발생할 가능성이 높은 부분을 위협 모델링 양식에 맞추어 정리한 것임.
컨트랙트 버전: `pragma solidity 0.8.26`

---

## 모듈: `BGT.sol`
### Function: `whitelistSender(address,bool)`
**Inputs**
- `sender`
  - **Control**: 오너(onlyOwner)
  - **Constraints**: 0 주소 불가
  - **Impact**: 토큰 전송 허용/차단
- `approved`
  - **Control**: 오너
  - **Constraints**: 불리언 값
  - **Impact**: whitelist 상태 토글

**Controls & Mitigations**
- `onlyOwner` 제어 → 멀티시그, 타임락 적용 권장
- 이벤트 모니터링 & 거버넌스 대시보드로 변경 사항 투명화

**Potential Negative Behavior / Abuse Scenarios**
- 오너 키 탈취 시 악의적 주소 화이트리스트 등록
- 결과적으로 토큰 이동 제한 우회 및 BGT를 외부 시장에서 거래 가능 → 부스트 경제 왜곡

**Impacts**
- **경제적 손실**: 부스트 토큰 가치를 희석시켜 검증자 인센티브 감소
- **거버넌스 영향**: 비정상 유통량 증대로 투표 결과 왜곡 가능

**Branches & Code Coverage**
- 분기: `sender == address(0)` 오류, `onlyOwner` 스킵 여부
- 테스트: (1) 일반 계정 호출 실패, (2) 오너 성공, (3) 0주소 입력 revert

### Function: `mint(address,uint256)`
**Inputs**
- `distributor`
  - **Control**: BlockRewardController (minter 역할)
  - **Constraints**: 0 주소 불가
  - **Impact**: 새 BGT 수령자
- `amount`
  - **Control**: BlockRewardController
  - **Constraints**: uint256, 상한 없음
  - **Impact**: 새로 발행될 BGT 양

**Controls & Mitigations**
- `onlyBlockRewardController` 제한 → 컨트롤러 주소를 멀티시그에 위임
- `invariantCheck`로 총 공급 ≈ 계약 ETH 예치량 검증

**Potential Negative Behavior / Abuse Scenarios**
- 컨트롤러 업그레이드/키 탈취 시 무제한 발행 가능
- 발행 후 `distributor`가 스테이킹하지 않고 즉시 리딤 → 계약 ETH 고갈

**Impacts**
- **경제적 손실**: 무한 인플레이션 & 계약 ETH 유출
- **무결성**: 총 공급 ≠ 회계 잔고 → 회계 불일치

**Branches & Code Coverage**
- 분기: (1) `distributor==0` revert, (2) 인플레 체크 실패 시 invariant revert
- 테스트: 최대 값 발행·리디밍 시 가스/상태 검사

### Function: `redeem(address,uint256)`
**Inputs**
- `receiver`
  - **Control**: 외부 호출자
  - **Constraints**: 0 주소 허용
  - **Impact**: ETH 수령 계정
- `amount`
  - **Control**: 외부 호출자
  - **Constraints**: `checkUnboostedBalance`로 잔액 ≤ 언부스트 토큰
  - **Impact**: 소각·출금할 BGT/ETH

**Controls & Mitigations**
- 토큰과 ETH 잔액 매핑 검증(`invariantCheck`) 필요
- ReentrancyGuard 없음 → `safeTransferETH` 후 재진입 위험 분석 필요

**Potential Negative Behavior / Abuse Scenarios**
- 공급량보다 많거나 ETH 부족 시 거래 실패→ DOS 가능성
- 재진입 공격으로 `emit Redeem` 이전 상태 조작 시 회계 오류

**Impacts**
- **가용성**: ETH 부족 시 모든 리딤 차단
- **경제적 손실**: 재진입으로 이중 출금 가능

**Branches & Code Coverage**
- 분기: (1) `checkUnboostedBalance` 통과 여부, (2) ETH 전송 성공/실패
- 테스트: 재진입 시나리오, ETH 잔액 0일 때 리딤 시 revert

## 모듈: `BGTStaker.sol`
### Function: `initialize(...)`
**Inputs**
- `_bgt, _feeDistributor, _governance, _rewardToken`
  - **Control**: 배포자 (owner)
  - **Constraints**: 0 주소 불가
  - **Impact**: 핵심 참조 주소 초기화

**Controls & Mitigations**
- `initializer` 한번만 실행 → 재초기화 불가 테스트 포함
- 파라미터 검증 강화(EOA/컨트랙트 여부)

**Potential Negative Behavior / Abuse Scenarios**
- 초기화 시 잘못된 컨트랙트 주소 입력 → 토큰 손실, 업그레이드 영구 고립
- 사기성 배포로 임의의 `_governance` 선택 시 거버넌스 하이재킹

**Impacts**
- **무결성**: 잘못된 주소로 리워드 계산 오류
- **가용성**: 업그레이드 거부로 프로토콜 중단

**Branches & Code Coverage**
- 분기: `initializer` 호출 횟수, 각 주소 0검사
- 테스트: 재초기화 시 revert, 0주소 입력 시 revert

### Function: `notifyRewardAmount(uint256)`
**Inputs**
- `reward`
  - **Control**: Owner
  - **Constraints**: ≠0, ≤ 보유량
  - **Impact**: Staking 보상 풀 증가

**Controls & Mitigations**
- `onlyOwner` → 멀티시그 적용
- `checkRewardSolvency` 내부 호출로 잔액 검증

**Potential Negative Behavior / Abuse Scenarios**
- 타이밍 공격: 보상 토큰량 증가 후 막대한 스테이크 직후 출금 → 보상 러그풀
- 오너가 과대 보상 설정 후 회수, 과거 사용자 보상 희석

**Impacts**
- **경제적 손실**: 보상 토큰 고갈 시 getReward() 실패
- **신뢰도**: 사용자 ROI 예측 불가

**Branches & Code Coverage**
- 분기: reward > stakingTokenBalance? revert
- 테스트: 대량 스테이크/보상 조합, uint 오버플로 테스트

### Function: `recoverERC20(address,uint256)`
**Inputs**
- `tokenAddress`
  - **Control**: Owner
  - **Constraints**: != rewardToken
  - **Impact**: 회수할 토큰 주소
- `tokenAmount`
  - **Control**: Owner
  - **Constraints**: 任意
  - **Impact**: 회수량

**Controls & Mitigations**
- 토큰 종류 제한만 존재 → 스테이크 토큰/BGT 회수 가능성 없음 확인
- 이벤트 로깅 및 옵저버 알림

**Potential Negative Behavior / Abuse Scenarios**
- Owner가 실수로 스테이킹 토큰 회수 시 사용자 예치금 동결
- Governance key 탈취 시 프로토콜 러그

**Impacts**
- **경제적 손실**: 유동성 손실, 사용자 예치금 미상환
- **가용성**: 회수 이후 보상 계산 실패 가능

**Branches & Code Coverage**
- 분기: tokenAddress==rewardToken? revert
- 테스트: 회수 후 getReward() 영향 테스트

### Function: `stake/withdraw(address,uint256)`
**Inputs**
- `account`
  - **Control**: BGT 컨트랙트(onlyBGT)
  - **Constraints**: 0 주소 불가
  - **Impact**: 스테이크 대상
- `amount`
  - **Control**: BGT 컨트랙트
  - **Constraints**: >0, uint128?
  - **Impact**: 예치/출금 양

**Controls & Mitigations**
- 호출자가 BGT에 한정 → BGT가 화이트리스트 해제 시 우회 가능 여부 검토
- `_safeTransferFromStakeToken`에서 transferFrom 사용 → allowance==∞ 예상

**Potential Negative Behavior / Abuse Scenarios**
- BGT 컨트랙트 또는 릴레이가 악용 시 강제 스테이크/언스테이크 가능
- Transfer 재진입으로 rewardDebt 조작 가능

**Impacts**
- **무결성**: 잘못된 보상 분배
- **경제적 손실**: 사용자가 의도치 않게 토큰 묶임

**Branches & Code Coverage**
- 분기: amount==0? noop, balance overflow 체크
- 테스트: 재진입, 허위 account 값 입력 공격

## 모듈: `StakingRewards.sol (내부 로직)`
### Function: `_notifyRewardAmount(uint256)`
**Inputs**
- `reward`
  - **Control**: 상속 컨트랙트(BGTStaker)
  - **Constraints**: ≤ 보유량
  - **Impact**: 리워드 풀 증가

**Controls & Mitigations**
- `_checkRewardSolvency()` 호출 → 실제 잔액 >= 계획 보상 확인
- 리워드 토큰 decimal 차이/변조 검증 필요

**Potential Negative Behavior / Abuse Scenarios**
- 리워드 토큰(ERC20) 가짜 전송 후 반환 → 보상 커밋 후 스왑하여 고갈 시도
- Miner extractable front‑run: rewardRate ↑ 직후 대량 스테이크

**Impacts**
- **경제적 손실**: 보상 언밸런스, APR 급변
- **공정성**: 새 스테이커가 과도 보상 수취

**Branches & Code Coverage**
- 분기: `rewardRate` 계산 분모 0 확인, 남은 보상 <0 revert
- 테스트: timestamp 조작, 블록 스킵 후 stake

### Function: `_setRewardsDuration(uint256)`
**Inputs**
- `_rewardsDuration`
  - **Control**: 상속 컨트랙트 Owner
  - **Constraints**: >0
  - **Impact**: 보상 주기 길이

**Controls & Mitigations**
- `rewardPeriodFinish` 경과 후만 변경 가능 → DOS 위험 낮음
- 시간 단위 설정 오류 방지

**Potential Negative Behavior / Abuse Scenarios**
- 매우 짧게 설정 시 APR 폭등 → 리워드 급배출, Front‑run 유도
- 매우 길게 설정 시 장기간 파라미터 동결

**Impacts**
- **경제적 손실**: 리워드 풀 고갈/동결
- **정책 리스크**: Governance 투표 간 시간차 증대

**Branches & Code Coverage**
- 분기: block.timestamp > rewardPeriodFinish 필수
- 테스트: edge duration 1, large duration overflow


---

## 모듈: `RewardVault.sol (내부 로직)`
### Function: `initialize(...)`
**Inputs**
- `_beaconDepositContract`
  - **Control**: 배포자 (Factory)
  - **Constraints**: 0주소 불가, 컨트랙트 여부 확인
  - **Impact**: BeaconDeposit 참조
- `_bgt`
  - **Control**: 배포자
  - **Constraints**: BGT 토큰 주소
  - **Impact**: 스테이킹 자산 참조
- `_distributor`
  - **Control**: 배포자
  - **Constraints**: Distributor 컨트랙트
  - **Impact**: 보상 알림 권한
- `_stakingToken`
  - **Control**: 배포자
  - **Constraints**: ERC20
  - **Impact**: Stake 토큰 주소

**Controls & Mitigations**
- `initializer` only once → 오프체인 배포 스크립트 검증
- 각 주소에 코드 사이즈 확인 및 EOA 차단
- 멀티시그/타임락 배포 권장

**Potential Negative Behavior / Abuse Scenarios**
- 잘못된 주소(피싱 컨트랙트) 지정 시 보상 분배·스테이킹 자산 손실
- 중복 초기화 취약점 CVE‑2023‑____ 유사 시나리오

**Impacts**
- **무결성**: 잘못된 참조로 함수 호출 revert, 사용자 자산 잠금
- **가용성**: 업그레이드 불가로 프로토콜 중단

**Branches & Code Coverage**
- 분기: initializer flag, 각 파라미터 0 검사
- 테스트: 이중 초기화, 잘못된 주소 입력 revert

---
### Function: `notifyRewardAmount(bytes,uint256)`
**Inputs**
- `pubkey`
  - **Control**: Distributor
  - **Constraints**: 32byte 이상
  - **Impact**: 검증자 식별
- `reward`
  - **Control**: Distributor
  - **Constraints**: uint256, 토큰 잔액 이하
  - **Impact**: 보상 규모

**Controls & Mitigations**
- `onlyDistributor` 제어 → Distributor 주소 멀티시그화
- `_notifyRewardAmount` 내부 잔액≥reward 검사, ReentrancyGuard 적용

**Potential Negative Behavior / Abuse Scenarios**
- Distributor 키 탈취로 보상 조작 및 인센티브 발행 우회
- `_processIncentives` 호출 중 외부 콜 → 재진입/상태 경합

**Impacts**
- **경제적 손실**: 과다 보상으로 인센티브 토큰 방출·인플레
- **무결성**: rewardPerToken 왜곡으로 APR 오류

**Branches & Code Coverage**
- 분기: reward==0? noop, StakeToken balance 부족시 revert
- 테스트: 재진입 모킹, 최대 reward, pubkey 변조

---
### Function: `whitelistIncentiveToken(address,uint256,address)`
**Inputs**
- `token`
  - **Control**: FactoryOwner
  - **Constraints**: ERC20, 0주소금지
  - **Impact**: 인센티브 토큰 지정
- `minIncentiveRate`
  - **Control**: FactoryOwner
  - **Constraints**: 1…MAX_INCENTIVE_RATE
  - **Impact**: 최소 인센티브 비율
- `manager`
  - **Control**: FactoryOwner
  - **Constraints**: EOA/컨트랙트
  - **Impact**: 토큰 관리자 권한

**Controls & Mitigations**
- `onlyFactoryOwner` → 타임락 적용
- 토큰 갯수 상한 `maxIncentiveTokensCount` 설정

**Potential Negative Behavior / Abuse Scenarios**
- 악성 토큰 추가 후 Manager가 `transferFrom`으로 예치 인센티브 탈취
- 지나치게 높은 `minIncentiveRate` 설정으로 다른 인센티브 토큰 효과 차단

**Impacts**
- **경제적 손실**: 정직한 프로토콜의 인센티브 참여 기회 박탈
- **정책 리스크**: 인센티브 시장 독점

**Branches & Code Coverage**
- 분기: minRate==0 revert, >MAX revert, token 이미 리스트 revert
- 테스트: 경계값·리밸런스·토큰 limit

---
### Function: `removeIncentiveToken(address)`
**Inputs**
- `token`
  - **Control**: FactoryVaultManager
  - **Constraints**: 화이트리스트된 토큰
  - **Impact**: 인센티브 토큰 제거

**Controls & Mitigations**
- `onlyFactoryVaultManager` ↔ 운영자 권한 분리 요구
- 토큰 제거 후 남은 보상 처리 로직 필요

**Potential Negative Behavior / Abuse Scenarios**
- Manager가 경쟁사 토큰 강제 제거 → 사용자 예치 인센티브 미회수, APR 급감
- 토큰 제거 직후 프런트런하여 남은 토큰 인출

**Impacts**
- **경제적 손실**: 사용자 보상 손실, 토큰 가격 급락
- **신뢰도**: 인센티브 프로그램 불확실성

**Branches & Code Coverage**
- 분기: onlyWhitelistedToken modifier, delete 후 length 감소 확인
- 테스트: remove 후 getWhitelistedTokensCount

---
### Function: `recoverERC20(address,uint256)`
**Inputs**
- `tokenAddress`
  - **Control**: FactoryOwner
  - **Constraints**: !=stakeToken, !=whitelistedIncentives
  - **Impact**: 회수 토큰
- `tokenAmount`
  - **Control**: FactoryOwner
  - **Constraints**: 任意<=잔액
  - **Impact**: 회수량

**Controls & Mitigations**
- 회수 제한 조건 강화: 예치 토큰·인센티브 토큰 제외
- 이벤트 모니터링·멀티시그

**Potential Negative Behavior / Abuse Scenarios**
- 오너가 실수로 StakeToken 조건 우회 업그레이드 후 회수 → 러그풀
- 프런트런으로 오너 회수 직전 사용자 예치→결제 실패(DOS)

**Impacts**
- **경제적 손실**: Vault 잔액 감소, 보상 불가
- **가용성**: getReward 변환 실패

**Branches & Code Coverage**
- 분기: 조건 위반 revert, safeTransfer 성공 여부
- 테스트: 회수 후 stake/withdraw 정상 여부

---
### Function: `stake/withdraw/getReward`
**Inputs**
- `amount`
  - **Control**: 사용자
  - **Constraints**: >0, allowance 충족
  - **Impact**: 예치·출금·보상 수령량
- `account/recipient`
  - **Control**: 사용자/Operator
  - **Constraints**: 주소 유효
  - **Impact**: 보상 수령자

**Controls & Mitigations**
- `ReentrancyGuard`, `whenNotPaused` 적용
- `_updateReward` 호출 순서 검증
- Operator 패턴→허위 operator 승인 방지

**Potential Negative Behavior / Abuse Scenarios**
- Flash‑loan 기반 대량 stake 후 즉시 getReward → 이득 확보 (Yield Draining)
- Operator 키 탈취로 타 계정 보상 탈취
- 재진입 Gas stipend 우회 실험

**Impacts**
- **경제적 손실**: 보상 풀 고갈, 정직한 사용자 APR 감소
- **무결성**: rewardPerToken 계산 오류

**Branches & Code Coverage**
- 분기: checkSelfStakedBalance modifier, zero amount skip
- 테스트: flash‑stake, operator override, reentrancy fuzz

---

*이 보고서는 업로드된 Solidity 소스 및 Markdown 문서를 기반으로 작성되었으며, 테스트 커버리지는 추정치입니다. 반드시 실제 단위·통합 테스트로 검증하세요.*
