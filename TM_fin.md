# BearMoon 프로젝트 전체 위협 모델

다음 문서는 Berachain 기반 BearMoon 프로젝트의 모든 스마트 컨트랙트에 대한 위협 모델을 **폴더별**, **함수별**로 정리한 Markdown 문서입니다.

---

## 1. Base 디렉터리

### 1.1 Create2Deployer.sol

#### Function: `deploy(bytes32 salt, bytes memory bytecode)`

**이 함수는 CREATE2를 사용해 새로운 컨트랙트를 배포합니다.**

- **Inputs**

  - `salt`
    - **Control:** 완료 호출자 제어
    - **Constraints:** 32바이트 값. 동일 salt로는 중복 배포 불가.
    - **Impact:** 결과 컨트랙트 주소 결정
  - `bytecode`
    - **Control:** 호출자 제어
    - **Constraints:** 유효한 EVM 바이트코드여야 함
    - **Impact:** 배포되는 로직

- **Branches and code coverage**

  - **정상 경로**: `bytecode.length > 0` → `create2` 호출 → 컨트랙트 주소 반환 → 이벤트 `Deployed(address, salt)` 발행
  - **오류 경로**: `bytecode.length == 0` 시 revert

- **Negative behavior**

  - 공격자가 악성 바이트코드를 전달해 시스템에 해로운 로직을 배포할 수 있음.

- **Negative test**

  - `deploy(salt, "")` 호출 시 항상 revert.

---

### 1.2 FactoryOwnable.sol

#### Function: `transferOwnership(address newOwner)`

**이 함수는 컨트랙트 소유권을 새 주소로 이전합니다.**

- **Inputs**

  - `newOwner`
    - **Control:** 현재 `owner`만 제어
    - **Constraints:** `newOwner != address(0)`
    - **Impact:** `owner` 상태 변경

- **Branches and code coverage**

  - **정상 경로**: `msg.sender == owner` → `owner = newOwner` → 이벤트 `OwnershipTransferred` 발행
  - **오류 경로**: 권한 체크 실패 시 revert

- **Negative behavior**

  - 권한 탈취: 악의적 `owner`가 자신으로 이전

- **Negative test**

  - 제3자(`msg.sender != owner`)가 호출 시 항상 revert.

---

### 1.3 IStakingRewards.sol / IStakingRewardsErrors.sol

인터페이스 및 에러 선언만 포함하고 있어 실행 로직이 없으므로 Threat Model에서 제외합니다.

---

### 1.4 StakingRewards.sol

#### Function: `stake(uint256 amount)`

**이 함수는 사용자가 스테이킹 풀에 BGT를 예치합니다.**

- **Inputs**

  - `amount`
    - **Control:** 호출자 제어
    - **Constraints:** `amount > 0` && `amount <= balanceOf(msg.sender)`
    - **Impact:** 스테이킹된 토큰 양 증가

- **Branches and code coverage**

  - **정상 경로**: `amount` 검증 통과 → `totalSupply += amount` && `balances[msg.sender] += amount` → `transferFrom(msg.sender, address(this), amount)` → 이벤트 `Staked(msg.sender, amount)` 발행
  - **오류 경로**: `amount == 0` 또는 `amount > balance` 시 revert

- **Negative behavior**

  - DoS: 매우 작은 단위로 반복 스테이킹해 가스 비용 증가 유발 가능

- **Negative test**

  - `stake(0)` 또는 `stake(exceedBalance)` 호출 시 항상 revert.

#### Function: `withdraw(uint256 amount)`

**이 함수는 사용자가 스테이킹한 BGT를 인출합니다.**

- **Inputs**

  - `amount`
    - **Control:** 호출자 제어
    - **Constraints:** `amount > 0` && `amount <= balances[msg.sender]`
    - **Impact:** 인출된 토큰 양 감소

- **Branches and code coverage**

  - **정상 경로**: 검증 통과 → `totalSupply -= amount` && `balances[msg.sender] -= amount` → `transfer(msg.sender, amount)` → 이벤트 `Withdrawn(msg.sender, amount)` 발행
  - **오류 경로**: `amount == 0` 또는 `amount > balance` 시 revert

- **Negative behavior**

  - Reentrancy: 외부 호출 후 상태 업데이트 순서가 잘못될 경우 재진입 공격 가능

- **Negative test**

  - `withdraw(0)` 또는 `withdraw(exceedBalance)` 호출 시 항상 revert.

#### Function: `getReward()`

**이 함수는 사용자의 누적 보상을 지급합니다.**

- **Inputs**

  - 없음 (전송 대상은 `msg.sender`)

- **Branches and code coverage**

  - **정상 경로**: `reward = earned(msg.sender)` 계산 → `require(reward > 0)` → `rewardToken.transfer(msg.sender, reward)` → 이벤트 `RewardPaid(msg.sender, reward)` 발행
  - **오류 경로**: `reward == 0` 시 revert

- **Negative behavior**

  - Spoofing: reward 계산 로직 오버플로우를 유발해 부정 이득 취득 시도

- **Negative test**

  - `getReward()` 호출 시 `earned == 0` 이면 항상 revert.

#### Function: `exit()`

**이 함수는 스테이킹 해제 및 보상 청구를 한 번에 처리합니다.**

- **Inputs**

  - 없음

- **Branches and code coverage**

  - **정상 경로**: 내부에서 `withdraw(balances[msg.sender])` 호출 후 `getReward()` 호출
  - **오류 경로**: 둘 중 하나라도 revert 시 전체 트랜잭션 revert

- **Negative behavior**

  - 복합 호출 과정에서 한 경로 실패 시 전체 실패로 인한 UX 저하

- **Negative test**

  - 잔액이 0인 사용자가 `exit()` 호출 시 항상 revert.

---

## 2. Extras (Oracles) 디렉터리

### 2.1 IPriceOracle.sol

- 인터페이스만 정의하므로 Threat Model 생략

### 2.2 IRootPriceOracle.sol

- 인터페이스만 정의하므로 Threat Model 생략

### 2.3 PeggedPriceOracle.sol

#### Function: `getPrice(address base, address quote)`

**이 함수는 고정 환율(oempegged)으로 설정된 두 토큰 간의 가격을 반환합니다.**

- **Inputs**

  - `base`
    - **Control:** 호출자 제어
    - **Constraints:** 사전 구성된 페어 중 하나여야 함
    - **Impact:** 조회 대상 베이스 토큰
  - `quote`
    - **Control:** 호출자 제어
    - **Constraints:** 사전 구성된 페어 중 하나여야 함
    - **Impact:** 조회 대상 쿼트 토큰

- **Branches and code coverage**

  - **정상 경로**: `(base, quote)` 페어가 매핑에 존재할 경우 `rates[base][quote]` 반환 → 이벤트 없음
  - **오류 경로**: 페어가 미등록 시 `revert UnsupportedPair(base, quote)`

- **Negative behavior**

  - 잘못된 페어로 지속 조회 시 반복 revert로 DoS 유발
  - 페어 매핑 오염 시 허위 가격 제공

- **Negative test**

  - `getPrice(unlistedBase, listedQuote)` 호출 시 `UnsupportedPair`로 revert
  - `getPrice(listedBase, unlistedQuote)` 호출 시 `UnsupportedPair`로 revert

---

### 2.4 PythPriceOracle.sol

#### Function: `updatePriceFeeds(bytes32[] feedIds, bytes[] updateData)`

**이 함수는 Pyth 네트워크로부터 가격 피드 데이터를 받아 로컬에 업데이트합니다.**

- **Inputs**

  - `feedIds`
    - **Control:** 호출자 제어
    - **Constraints:** 배열 길이 일치, 각 ID는 사전 허용된 key여야 함
    - **Impact:** 업데이트 대상 피드 식별자 목록
  - `updateData`
    - **Control:** 호출자 제어
    - **Constraints:** `updateData.length == feedIds.length`, SSVM 시리얼라이즈된 데이터여야 함
    - **Impact:** 각 피드별 새 가격 데이터

- **Branches and code coverage**

  - **정상 경로**: owner 또는 허가된 updater만 호출가능 → 내부 `_validateAndApply(feedId, data)` 반복 → `PriceUpdated(feedId, newPrice)` 이벤트 발행
  - **오류 경로**: 배열 길이 불일치 시 `revert InvalidInputLength()`; 비허가자 호출 시 `revert Unauthorized()`; SSVM 검증 실패 시 `revert InvalidPriceData(feedId)`

- **Negative behavior**

  - 잘못된 `updateData`로 SSVM 검증 우회 시 허위 가격 반영
  - 다수 feedIds 전달해 가스 소진으로 인한 DoS
  - 권한 체계 우회 시 외부 공격자가 가격 악의 조작

- **Negative test**

  - `updatePriceFeeds([A], [badData])` 호출 시 `InvalidPriceData(A)`로 revert
  - 비허가자(`msg.sender != owner`) 호출 시 `Unauthorized()`로 revert

#### Function: `consult(address base, address quote) view returns (uint256 price)`

**이 함수는 Pyth에 저장된 두 자산 간의 실시간 가격을 반환합니다.**

- **Inputs**

  - `base`, `quote`
    - **Control:** 호출자 제어
    - **Constraints:** `feedIds` 매핑에 사전 등록된 asset ID 여야 함
    - **Impact:** 조회 대상 자산 페어

- **Branches and code coverage**

  - **정상 경로**: feed 매핑 조회 → 현재 `price = latestPrice[base][quote]` 반환
  - **오류 경로**: 조회 매핑이 없으면 `revert UnknownAssetPair(base, quote)`

- **Negative behavior**

  - 코드상 등록되지 않은 페어로 조회 시 반복 오류 유발
  - feed 업데이트 지연으로 stale price 제공

- **Negative test**

  - `consult(unregisteredBase, quote)` 호출 시 `UnknownAssetPair`로 revert
  - `consult(base, unregisteredQuote)` 호출 시 `UnknownAssetPair`로 revert

---

### 2.5 PythPriceOracleDeployer.sol

#### Function: `deploy(address pythAddress, bytes32[] initialFeedIds)`

**이 함수는 PythPriceOracle 컨트랙트를 배포하고 초기 가격 피드를 설정합니다.**

- **Inputs**

  - `pythAddress`
    - **Control:** 호출자 제어
    - **Constraints:** 유효한 Pyth 네트워크 컨트랙트 주소
    - **Impact:** 내부 oracle 레퍼런스
  - `initialFeedIds`
    - **Control:** 호출자 제어
    - **Constraints:** 사전 검증된 feed list
    - **Impact:** 초기 등록 피드 목록

- **Branches and code coverage**

  - **정상 경로**: 생성자에서 `new PythPriceOracle(pythAddress)` 호출 → `initializeFeeds(initialFeedIds)` → 이벤트 `DeployerSet(firstOracle)` 발행
  - **오류 경로**: `pythAddress == address(0)` 시 revert; 빈 `initialFeedIds` 배열 허용 여부에 따른 분기

- **Negative behavior**

  - 잘못된 `pythAddress` 전달로 oracle 무효화
  - `initialFeedIds` 조작 시 서비스 초기화 오류

- **Negative test**

  - `deploy(address(0), [A,B])` 호출 시 항상 revert

---

### 2.6 RootPriceOracle.sol

#### Function: `setRoot(bytes32 newRoot)`

**이 함수는 루트 컨트랙트에 새로운 Merkle root를 설정합니다.**

- **Inputs**

  - `newRoot`
    - **Control:** owner 또는 governor 제어
    - **Constraints:** 32바이트 SSZ-encoded Merkle root
    - **Impact:** 향후 증명 검증 기준

- **Branches and code coverage**

  - **정상 경로**: `onlyOwner` 체크 통과 → `currentRoot = newRoot` → 이벤트 `RootUpdated(newRoot)` 발행
  - **오류 경로**: 비허가자 호출 시 `revert Unauthorized()`

- **Negative behavior**

  - 악의적 root 설정으로 잘못된 Merkle 증명 통과 유도

- **Negative test**

  - 제3자 호출 시 항상 revert

#### Function: `readRoot() view returns (bytes32)`

**이 함수는 현재 저장된 Merkle root를 반환합니다.**

- **Inputs**

  - 없음

- **Branches and code coverage**

  - 항상 `currentRoot` 반환

- **Negative behavior**

  - 없음 (pure view)

- **Negative test**

  - 없음

---

### 2.7 RootPriceOracleDeployer.sol

#### Function: `deployRootOracle(address owner)`

**이 함수는 RootPriceOracle을 배포하고 초기 소유권을 설정합니다.**

- **Inputs**

  - `owner`
    - **Control:** 호출자 제어
    - **Constraints:** non-zero address
    - **Impact:** 배포된 컨트랙트의 `owner`

- **Branches and code coverage**

  - **정상 경로**: CREATE2 또는 `new` 키워드로 `RootPriceOracle` 인스턴스 생성 → `transferOwnership(owner)` → 이벤트 `RootOracleDeployed(address, owner)` 발행
  - **오류 경로**: `owner == address(0)` 시 revert

- **Negative behavior**

  - 잘못된 owner 전달로 권한 회수 불능

- **Negative test**

  - `deployRootOracle(address(0))` 호출 시 항상 revert

---

## 3. Governance 디렉터리

### 3.1 BerachainGovernance.sol

#### Function: `propose(bytes calldata proposalData)`

**이 함수는 새로운 거버넌스 제안(proposal)을 생성합니다.**

- **Inputs**

  - `proposalData`
    - **Control:** 호출자 제어
    - **Constraints:** EVM 인코딩된 함수 호출 리스트와 실행 조건을 포함해야 함
    - **Impact:** 생성될 프로포절의 실행 로직 및 대상

- **Branches and code coverage**

  - **정상 경로**: `msg.sender`가 권한 있는 제안자일 경우 → 내부 `_validateProposal(proposalData)` → proposals 맵에 저장 → 이벤트 `ProposalCreated(id, proposer)` 발행
  - **오류 경로**: 권한 없거나 입력 형식 오류 시 `revert InvalidProposal()` 또는 `revert Unauthorized()`

- **Negative behavior**

  - 악의적 제안 생성: 악성 코드를 포함한 proposalData를 제출해 추후 실행 시 시스템 훼손
  - DoS: 반복적 소규모 Proposal 생성으로 스토리지 및 가스 소진

- **Negative test**

  - 권한 없는 주소가 호출 시 반드시 `Unauthorized()`로 revert
  - 잘못된 데이터 형식 제출 시 반드시 `InvalidProposal()`로 revert

---

#### Function: `vote(uint256 proposalId, bool support)`

**이 함수는 지정된 프로포절에 대하여 찬반 투표를 실시합니다.**

- **Inputs**

  - `proposalId`
    - **Control:** 호출자 제어
    - **Constraints:** 존재하는 proposalId여야 함
    - **Impact:** 투표 적용 대상 proposal
  - `support`
    - **Control:** 호출자 제어
    - **Constraints:** `true` 또는 `false`
    - **Impact:** 찬성/반대 여부

- **Branches and code coverage**

  - **정상 경로**: proposal 기간 내 투표 시 → `votes[proposalId][msg.sender] = support` → 이벤트 `VoteCast(msg.sender, proposalId, support)`
  - **오류 경로**: 기간 외 투표, 중복 투표, 존재하지 않는 proposalId 시 `revert VoteNotAllowed()` 또는 `revert InvalidProposalId()`

- **Negative behavior**

  - 중복 투표 우회: 투표 기록 초기화 후 반복 투표 시도
  - 투표 기간 조작: proposal 생성자 권한 탈취 시 유리한 타이밍에 vote 호출

- **Negative test**

  - 동일 주소가 두 번 vote() 호출 시 두 번째 호출 반드시 `VoteNotAllowed()`로 revert
  - 기간 지난 proposalId로 vote() 호출 시 반드시 `VoteNotAllowed()`로 revert

---

#### Function: `execute(uint256 proposalId)`

**이 함수는 통과된 프로포절을 실행합니다.**

- **Inputs**

  - `proposalId`
    - **Control:** 호출자 제어
    - **Constraints:** 투표 결과 통과된 proposalId여야 함
    - **Impact:** 실행될 로직

- **Branches and code coverage**

  - **정상 경로**: `quorum` 및 `voteCount` 조건 충족 확인 → `proposalData` 순차 실행 → 이벤트 `ProposalExecuted(proposalId)`
  - **오류 경로**: 미통과 또는 중복 실행 시 `revert ExecutionNotAllowed()` 또는 `revert AlreadyExecuted()`

- **Negative behavior**

  - 중복 실행: `executed` 플래그 우회 시 두 번 실행 유도
  - DoS: 실행 시 가스 소진시키는 복잡한 proposalData 제출

- **Negative test**

  - 통과되지 않은 proposalId로 execute() 호출 시 반드시 `ExecutionNotAllowed()`로 revert
  - 동일 proposalId로 두 번 execute() 호출 시 반드시 `AlreadyExecuted()`로 revert

---

### 3.2 GovDeployer.sol

#### Function: `deployGovernance(address timelock, address[] memory initialGovernors)`

**이 함수는 거버넌스 모듈과 Timelock 컨트랙트를 배포/연결합니다.**

- **Inputs**

  - `timelock`
    - **Control:** 호출자 제어
    - **Constraints:** 유효한 Timelock 컨트랙트 주소
    - **Impact:** governance 권한 위임 대상
  - `initialGovernors`
    - **Control:** 호출자 제어
    - **Constraints:** 비어있지 않은 배열, 각 주소는 non-zero
    - **Impact:** 초기 거버넌서 리스트

- **Branches and code coverage**

  - **정상 경로**: `new Governance(timelock)` 배포 → `setGovernors(initialGovernors)` 호출 → 이벤트 `GovernanceDeployed(address)` 발행
  - **오류 경로**: `timelock == address(0)` 또는 `initialGovernors.length == 0` 시 revert

- **Negative behavior**

  - 잘못된 Timelock 주소 지정으로 governance 통제 상실
  - 초기 거버넌서 배열 조작으로 권한 부여 대상 왜곡

- **Negative test**

  - `deployGovernance(address(0), [A,B])` 호출 시 revert
  - `deployGovernance(timelock, [])` 호출 시 revert

---

### 3.3 TimeLock.sol

#### Function: `schedule(address target, bytes calldata data, uint256 eta)`

**이 함수는 Timelock 실행 대기열에 함수를 등록합니다.**

- **Inputs**

  - `target`
    - **Control:** governor 제어
    - **Constraints:** non-zero address
    - **Impact:** 나중에 호출될 컨트랙트
  - `data`
    - **Control:** governor 제어
    - **Constraints:** 유효한 인코딩
    - **Impact:** 실행할 함수 및 파라미터
  - `eta`
    - **Control:** governor 제어
    - **Constraints:** 현재 시간보다 충분히 큰 timestamp
    - **Impact:** 실행 가능 시간

- **Branches and code coverage**

  - **정상 경로**: `require(eta >= block.timestamp + MIN_DELAY)` → 저장소에 queue 등록 → 이벤트 `CallScheduled(target, dataHash, eta)`
  - **오류 경로**: `eta` 미충족 시 `revert TimestampNotValid()`

- **Negative behavior**

  - 타임락 우회: `MIN_DELAY` 이하 eta로 schedule 시도
  - 스토리지 오버플로우 반복 등록

- **Negative test**

  - `schedule(target, data, now + MIN_DELAY - 1)` 호출 시 revert

---

#### Function: `execute(address target, bytes calldata data, uint256 eta)`

**이 함수는 Timelock에 등록된 함수를 실제로 실행합니다.**

- **Inputs**

  - `target`, `data`, `eta` (schedule과 동일)

- **Branches and code coverage**

  - **정상 경로**: `require(block.timestamp >= eta)` && `queued[target][dataHash][eta]` 존재 확인 → call target with data → 이벤트 `CallExecuted(target, dataHash, eta)` → 삭제 from queue
  - **오류 경로**: 미도달 eta, 미등록 호출 시 `revert CallNotReady()` 또는 `revert CallNotFound()`

- **Negative behavior**

  - 실행 시 가스 소진하도록 복잡한 data 전달
  - queue 상태 업데이트 누락 시 재실행 가능성

- **Negative test**

  - `execute`를 eta 이전에 호출 시 revert
  - schedule 하지 않은 data로 execute 호출 시 revert

---

#### Function: `cancel(address target, bytes calldata data, uint256 eta)`

**이 함수는 Timelock에 등록된 실행 대기를 취소합니다.**

- **Inputs**

  - `target`, `data`, `eta`

- **Branches and code coverage**

  - **정상 경로**: `queued[target][dataHash][eta]` 확인 → 삭제 → 이벤트 `CallCancelled(target, dataHash, eta)`
  - **오류 경로**: 미등록 호출 시 `revert CallNotFound()`

- **Negative behavior**

  - 취소 후에도 상태 삭제 실패 시 재실행 가능성

- **Negative test**

  - 존재하지 않는 queue에 cancel 호출 시 revert

---

## 4. Libraries 디렉터리

### 4.1 BeaconRoots.sol

#### Function: `appendRoot(bytes32 root)`

**이 함수는 새로운 Beacon 체인 루트를 저장합니다.**

- **Inputs**

  - `root`
    - **Control:** owner 또는 governance 제어
    - **Constraints:** 32바이트 SSZ root여야 함
    - **Impact:** 루트 목록에 추가

- **Branches and code coverage**

  - **정상 경로**: `roots.push(root)` → 이벤트 `RootAppended(root)` 발행
  - **오류 경로**: 없음 (push always succeeds)

- **Negative behavior**

  - 잘못된 root 입력 시 이후 검증 로직 무력화

- **Negative test**

  - 적절하지 않은 root라도 push 가능하므로, 검증 로직과 조합해 테스트 필요

---

#### Function: `verifyProof(bytes32 root, bytes calldata proof, bytes32 leaf) view returns (bool)`

**이 함수는 Merkle proof를 검증해 leaf가 root에 속하는지 확인합니다.**

- **Inputs**

  - `root`, `proof`, `leaf`
    - **Control:** 호출자 제어
    - **Constraints:** proof 포맷(해시 배열) 유효해야 함
    - **Impact:** proof 결과

- **Branches and code coverage**

  - **정상 경로**: 내부 `_computeRoot(leaf, proof)` == `root` → true 반환
  - **오류 경로**: 불일치 시 false 반환

- **Negative behavior**

  - 잘못 형성된 proof로 반복 연산 시 가스 소진 가능

- **Negative test**

  - 유효 증명 vs. 무효 증명 양쪽 모두 트리거

---

### 4.2 SSZ.sol

#### Function: `hashTreeRoot(bytes memory data) pure returns (bytes32)`

**이 함수는 SSZ 규격에 따라 data의 Merkle root를 계산합니다.**

- **Inputs**

  - `data`
    - **Control:** 호출자 제어
    - **Constraints:** SSZ 인코딩 규격 준수
    - **Impact:** 계산된 root

- **Branches and code coverage**

  - **정상 경로**: `serialize(data)` → `Merkleize(chunks)` → root 반환
  - **오류 경로**: 인코딩 불일치 시 계산 결과 잘못되거나 panic

- **Negative behavior**

  - 재귀적 대형 data 제공 시 스택/메모리 오버플로우 유발

- **Negative test**

  - 잘못 인코딩된 data에 대해 예상치 못한 오류 혹은 Panic 발생 검증

---

#### Function: `validateSSZ(bytes memory data) pure returns (bool)`

**이 함수는 data가 SSZ 규격을 준수하는지 검사합니다.**

- **Inputs**

  - `data`
    - **Control:** 호출자 제어
    - **Constraints:** SSZ 기본 형식
    - **Impact:** 유효성 검사 결과

- **Branches and code coverage**

  - **정상 경로**: 모든 필드 타입 및 길이 체크 통과 → true
  - **오류 경로**: 필드 불일치, 길이 초과 시 false

- **Negative behavior**

  - 복잡한 data로 반복 체크 시 가스 소진 가능

- **Negative test**

  - 유효 vs 무효 SSZ 데이터를 각각 테스트

---

### 4.3 Utils.sol

#### Function: `safeTransfer(IERC20 token, address to, uint256 amount)`

**이 함수는 ERC20 ****`transfer`**** 호출 후 결과를 검증해 실패 시 revert 합니다.**

- **Inputs**

  - `token`, `to`, `amount`
    - **Control:** 호출자 제어
    - **Constraints:** `to != address(0)`, `amount <= token.balanceOf(address(this))`
    - **Impact:** 토큰 전송

- **Branches and code coverage**

  - **정상 경로**: `token.transfer(to, amount)` 호출 → 반환값 또는 성공 확인
  - **오류 경로**: 반환값 false 또는 revert 시 `revert TransferFailed()`

- **Negative behavior**

  - gas stipend 부족으로 호출 실패 유도 가능

- **Negative test**

  - 부족한 잔액 혹은 zero address 전달시 revert 검증

---

#### Function: `checkAddress(address addr) pure returns (bool)`

**이 함수는 주소 유효성(Zero 주소 여부)만 검사합니다.**

- **Inputs**

  - `addr`
    - **Control:** 호출자 제어
    - **Constraints:** any address
    - **Impact:** bool 반환

- **Branches and code coverage**

  - **정상 경로**: `addr != address(0)` → true, else false
  - **오류 경로**: 없음

- **Negative behavior**

  - 없음

- **Negative test**

  - `checkAddress(0)` → false, `checkAddress(non-zero)` → true

---

---

## 5. PoL Rewards (pol/rewards 디렉터리)

### 5.1 BeraChef.sol

#### Function: `setVault(address vault, uint256 weight)`

**이 함수는 보상 분배 컨테이너(vault)를 등록하거나 가중치를 설정합니다.**

- **Inputs**
  - `vault`
    - **Control:** 오직 거버넌스 또는 owner가 제어
    - **Constraints:** `vault != address(0)` && VaultFactory에서 생성된 유효 vault여야 함
    - **Impact:** 분배 대상 리스트에 포함, 가중치 매핑 설정
  - `weight`
    - **Control:** 오직 거버넌스 또는 owner가 제어
    - **Constraints:** `weight > 0` && `weight <= MAX_WEIGHT`
    - **Impact:** 해당 vault의 보상 분배 비율

- **Branches and code coverage**
  - **정상 경로**: `onlyOwner` 혹은 `onlyGovernance` 체크 통과 → `vaultWeights[vault] = weight` → 이벤트 `VaultSet(vault, weight)` 발행
  - **오류 경로**: `vault == 0` 또는 `weight` 범위 벗어날 경우 `revert InvalidVaultOrWeight()`

- **Negative behavior**
  - 무효한 vault 주소를 등록하거나 weight를 과다 설정해 보상 왜곡
  - 가중치 0으로 설정해 특정 vault 제외

- **Negative test**
  - `setVault(address(0), 100)` 호출 시 반드시 revert
  - `setVault(validVault, 0)` 또는 `setVault(validVault, MAX_WEIGHT+1)` 호출 시 revert

---
#### Function: `distributeRewards()`

**이 함수는 누적된 BGT 보상을 모든 등록된 vault에 가중치 비율에 따라 분배합니다.**

- **Inputs**
  - 없음 (state에서 `totalRewards` 및 `vaultWeights` 참조)

- **Branches and code coverage**
  - **정상 경로**:
    1. `totalWeight = sum(vaultWeights)` 계산
    2. `totalRewards = balanceOf(this)` 읽기
    3. 각 `vault`마다 `share = totalRewards * weight / totalWeight` 계산 후 `safeTransfer(vault, share)`
    4. 이벤트 `RewardsDistributed(vault, share)` 반복 발행
  - **오류 경로**:
    - `totalWeight == 0` 시 `revert NoVaults()`
    - `totalRewards == 0` 시 `revert NoRewards()`

- **Negative behavior**
  - vault 수가 많아 가스 초과로 분배 중단(DoS)
  - 분배 도중 특정 vault에 대한 재진입 공격 가능

- **Negative test**
  - vault가 하나도 없을 때 호출 시 `NoVaults()`로 revert
  - 보상 잔액 0일 때 호출 시 `NoRewards()`로 revert

---

### 5.2 BGTIncentiveDistributor.sol

#### Function: `distributeToVaults()`

**이 함수는 인센티브 토큰을 등록된 RewardVault들로 분배합니다.**

- **Inputs**
  - 없음 (state에서 `incentiveToken` 및 `vaults` 참조)

- **Branches and code coverage**
  - **정상 경로**:
    1. `totalIncentives = incentiveToken.balanceOf(this)`
    2. 각 `vault`에 대해 `share = totalIncentives * allocation[vault] / totalAllocation` 계산
    3. `safeTransfer(vault, share)` → 이벤트 `IncentiveDistributed(vault, share)`
  - **오류 경로**: `totalAllocation == 0` 시 `revert NoAllocations()`; `totalIncentives == 0` 시 `revert NoIncentives()`

- **Negative behavior**
  - 불균형한 allocation 설정으로 특정 vault에 인센티브 집중
  - 반복 호출 시 동일 인센티브 중복 분배

- **Negative test**
  - allocation이 하나도 없을 때 호출 시 `NoAllocations()`로 revert
  - 인센티브 잔액 0일 때 호출 시 `NoIncentives()`로 revert

---

### 5.3 BlockRewardController.sol

#### Function: `computeBlockReward(uint256 blockNumber) view returns (uint256)`

**이 함수는 특정 블록 넘버 기준으로 보상량을 산출합니다.**

- **Inputs**
  - `blockNumber`
    - **Control:** 호출자 제어
    - **Constraints:** `blockNumber >= genesisBlock` && `blockNumber <= currentBlock` (view 함수이므로 revert 없을 수 있음)
    - **Impact:** 반환 보상량

- **Branches and code coverage**
  - **정상 경로**: `elapsed = blockNumber - genesisBlock` → `reward = baseReward * decayFactor^elapsed` 계산 후 반환
  - **오류 경로**: 통상 없음(view)

- **Negative behavior**
  - 매우 큰 `blockNumber` 입력 시 overflow 또는 gas 과다 소모 가능

- **Negative test**
  - `computeBlockReward(genesisBlock - 1)` 호출 시 정의된 동작(예: 0 반환 또는 revert) 확인

---
#### Function: `setParameters(uint256 newBaseReward, uint256 newDecayRate)`

**이 함수는 보상산출 파라미터를 업데이트합니다.**

- **Inputs**
  - `newBaseReward`
    - **Control:** 오직 거버넌스 제어
    - **Constraints:** `> 0`
    - **Impact:** 이후 보상 산출의 기본 값
  - `newDecayRate`
    - **Control:** 오직 거버넌스 제어
    - **Constraints:** `<= MAX_DECAY`
    - **Impact:** 보상 감소 계수

- **Branches and code coverage**
  - **정상 경로**: `onlyGovernance` 체크 통과 → 값 업데이트 → 이벤트 `ParamsUpdated(baseReward, decayRate)`
  - **오류 경로**: 요구조건 실패 시 `revert InvalidParams()`

- **Negative behavior**
  - `newDecayRate`를 매우 낮게 설정해 보상이 사실상 0으로 고갈
  - `newBaseReward`를 과도하게 높여 인플레이션 초래

- **Negative test**
  - `setParameters(0, X)` 호출 시 revert
  - `setParameters(Y, MAX_DECAY+1)` 호출 시 revert

---

### 5.4 Distributor.sol

#### Function: `addRecipient(address recipient, uint256 percent)`

**이 함수는 보상 분배 대상(recipient)을 추가하고 비율(percent)을 설정합니다.**

- **Inputs**
  - `recipient`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** `recipient != address(0)`
    - **Impact:** 분배 대상 리스트에 포함
  - `percent`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** `percent > 0` && 합계 `<= 100%
  - **Impact:** 해당 recipient의 분배 비율

- **Branches and code coverage**
  - **정상 경로**: 권한 체크 → `recipients.push(recipient)` 및 `percents[recipient]=percent` → 이벤트 `RecipientAdded(recipient, percent)`
  - **오류 경로**: invalid recipient 또는 percent 범위 실패 시 revert

- **Negative behavior**
  - 비정상 percent 설정(합계 초과)로 분배 불일치

- **Negative test**
  - `addRecipient(address(0), 10)` 호출 시 revert
  - `addRecipient(A, 0)` 호출 시 revert

---
#### Function: `removeRecipient(address recipient)`

**이 함수는 기존 recipient를 리스트에서 제거합니다.**

- **Inputs**
  - `recipient`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** 리스트에 존재해야 함
    - **Impact:** 리스트에서 삭제

- **Branches and code coverage**
  - **정상 경로**: 권한 체크 → 리스트에서 제거 → 이벤트 `RecipientRemoved(recipient)`
  - **오류 경로**: 존재하지 않는 recipient일 때 revert

- **Negative behavior**
  - 삭제 후 percent 합계 변화로 분배 계산 오류

- **Negative test**
  - `removeRecipient(notExist)` 호출 시 revert

---

### 5.5 RewardVaultFactory.sol

#### Function: `createVault(address owner)`

**이 함수는 새로운 RewardVault 컨트랙트를 배포하고 지정된 owner에게 소유권을 부여합니다.**

- **Inputs**
  - `owner`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** `owner != address(0)`
    - **Impact:** 배포된 Vault의 소유권

- **Branches and code coverage**
  - **정상 경로**: `new RewardVault()` 생성 → `transferOwnership(owner)` 호출 → 이벤트 `VaultCreated(vaultAddress, owner)`
  - **오류 경로**: `owner == address(0)` 시 revert

- **Negative behavior**
  - 잘못된 owner 전달로 잘못된 권한 설정

- **Negative test**
  - `createVault(address(0))` 호출 시 revert

---

### 5.6 RewardVault.sol

#### Function: `depositFees(address feeToken, uint256 amount)`

**이 함수는 FeeCollector로부터 feeToken을 받아 Vault에 보관합니다.**

- **Inputs**
  - `feeToken`
    - **Control:** 오직 VaultOwner 또는 allowList에 등록된 FeeCollector만 제어
    - **Constraints:** `feeToken`은 whitelist된 토큰
    - **Impact:** Vault에 보관될 토큰 종류
  - `amount`
    - **Control:** VaultOwner 또는 FeeCollector
    - **Constraints:** `amount > 0` && caller 보유 잔액 `>= amount`
    - **Impact:** Vault 내부 `balances[feeToken]` 증가

- **Branches and code coverage**
  - **정상 경로**: 권한 체크 → `transferFrom(caller, this, amount)` → `balances[feeToken] += amount` → 이벤트 `FeesDeposited(feeToken, amount)`
  - **오류 경로**: notWhitelisted token 또는 amount 조건 실패 시 revert

- **Negative behavior**
  - malicious feeToken으로 deposit 요청 시 잘못된 토큰 보관
  - 반복 deposit으로 gas consumption 증가

- **Negative test**
  - `depositFees(unlistedToken, 100)` 호출 시 revert
  - `depositFees(validToken, 0)` 호출 시 revert

---
#### Function: `withdrawFees(address feeToken, uint256 amount)`

**이 함수는 VaultOwner가 저장된 feeToken을 인출합니다.**

- **Inputs**
  - `feeToken`
    - **Control:** 오직 VaultOwner
    - **Constraints:** `balances[feeToken] >= amount`
    - **Impact:** Vault 내부 자산 감소
  - `amount`
    - **Control:** 오직 VaultOwner
    - **Constraints:** `amount > 0`
    - **Impact:** withdrawal amount

- **Branches and code coverage**
  - **정상 경로**: 권한 체크 → `balances[feeToken] -= amount` → `safeTransfer(caller, amount)` → 이벤트 `FeesWithdrawn(feeToken, amount)`
  - **오류 경로**: amount조건 또는 권한 실패 시 revert

- **Negative behavior**
  - 시점(Tx ordering) 조작으로 balances mismatch 가능
  - Reentrancy 공격

- **Negative test**
  - `withdrawFees(validToken, exceedBalance)` 호출 시 revert
  - non-owner 호출 시 revert

---

## 6. PoL Core 및 기타 주요 컨트랙트

### 6.1 BeaconDeposit.sol
#### Function: `deposit(bytes32 validatorRoot, uint256 amount) payable`

**이 함수는 Validator가 ETH를 스테이킹하고 Beacon 체인 루트를 제출합니다.**

- **Inputs**
  - `validatorRoot`
    - **Control:** 호출자 제어
    - **Constraints:** 32바이트 SSZ-encoded Merkle root여야 함
    - **Impact:** Validator의 최신 Beacon 상태 기록
  - `amount` (msg.value)
    - **Control:** 호출자 제어
    - **Constraints:** `amount == msg.value` && `amount >= MIN_DEPOSIT`
    - **Impact:** Validator의 지분 크기

- **Branches and code coverage**
  - **정상 경로**:
    1. `require(msg.value == amount, "ValueMismatch")`
    2. `require(amount >= MIN_DEPOSIT, "DepositTooSmall")`
    3. `deposits[msg.sender] = amount` && `roots[msg.sender] = validatorRoot`
    4. 이벤트 `Deposited(msg.sender, amount, validatorRoot)` 발행
  - **오류 경로**:
    - `msg.value != amount` → `ValueMismatch()` revert
    - `amount < MIN_DEPOSIT` → `DepositTooSmall()` revert

- **Negative behavior**
  - SSZ 파싱을 스마트컨트랙트에서 직접 수행할 경우, 대량 데이터로 DoS 가능
  - 잘못된 Merkle root로 PoL 검증 무력화 시도

- **Negative test**
  - `deposit(root, MIN_DEPOSIT - 1)` 호출 시 `DepositTooSmall()`로 revert
  - `deposit(root, value+1)` 호출 시 `ValueMismatch()`로 revert

---

### 6.2 BeaconRootsHelper.sol

#### Function: `submitBeaconRoot(bytes32 root)`

**이 함수는 새로운 Beacon Merkle root를 제출하여 보관합니다.**

- **Inputs**
  - `root`
    - **Control:** owner 또는 governance 제어
    - **Constraints:** 32바이트 SSZ Merkle root
    - **Impact:** 저장될 Beacon 루트

- **Branches and code coverage**
  - **정상 경로**: `onlyOwner` 체크 통과 → `roots.push(root)` → 이벤트 `RootSubmitted(root)` 발행
  - **오류 경로**: 권한 확인 실패 시 `revert Unauthorized()`

- **Negative behavior**
  - 악성 root 제출로 클라이언트의 PoL 검증 경로 우회
  - 과도한 root 제출로 스토리지 증가 유발

- **Negative test**
  - 비소유자 호출 시 반드시 `Unauthorized()`로 revert

---
#### Function: `verifyBeaconRoot(bytes32 root, bytes calldata proof, bytes32 leaf) view returns (bool)`

**이 함수는 Merkle proof를 통해 `leaf`가 `root`에 속하는지 검증합니다.**

- **Inputs**
  - `root`, `proof`, `leaf`
    - **Control:** 호출자 제어
    - **Constraints:** `proof`는 올바른 Merkle proof 형식
    - **Impact:** 반환 값(`true`/`false`)

- **Branches and code coverage**
  - **정상 경로**: 내부 `_computeRoot(leaf, proof) == root` → `return true`
  - **오류 경로**: 불일치 시 `return false`

- **Negative behavior**
  - 긴 proof 배열로 가스 소진 가능(DOS)

- **Negative test**
  - 올바른 proof vs. 임의의 proof 양쪽에 대해 `true`/`false` 반환 검증

---

### 6.3 BGT.sol / BGTFeeDeployer.sol / BGTStaker.sol

#### 6.3.1 BGT.sol

##### Function: `mint(address to, uint256 amount)`

**이 함수는 새로운 BGT 토큰을 발행합니다.**

- **Inputs**
  - `to`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** non-zero address
    - **Impact:** `to` 계정에 토큰 부여
  - `amount`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** `amount > 0`
    - **Impact:** 토큰 총공급 증가

- **Branches and code coverage**
  - **정상 경로**: 권한 체크 → `totalSupply += amount` && `balances[to] += amount` → 이벤트 `Transfer(address(0), to, amount)`
  - **오류 경로**: `to == address(0)` 또는 `amount == 0` 시 revert

- **Negative behavior**
  - 무제한 mint 요청으로 인플레이션 유발

- **Negative test**
  - `mint(address(0), 100)` 또는 `mint(user, 0)` 호출 시 revert


##### Function: `burn(address from, uint256 amount)`

**이 함수는 BGT 토큰을 소각합니다.**

- **Inputs**
  - `from`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** non-zero address, `balances[from] >= amount`
    - **Impact:** 해당 계정에서 토큰 소각
  - `amount`
    - **Control:** onlyOwner 또는 onlyGovernance
    - **Constraints:** `amount > 0`
    - **Impact:** 토큰 총공급 감소

- **Branches and code coverage**
  - **정상 경로**: 권한 체크 → `balances[from] -= amount` && `totalSupply -= amount` → 이벤트 `Transfer(from, address(0), amount)`
  - **오류 경로**: `amount == 0` або `balances[from] < amount` 시 revert

- **Negative behavior**
  - 악의적 소각 요청으로 정상 사용자 자산 소실

- **Negative test**
  - `burn(user, exceedBalance)` 호출 시 revert

---
#### 6.3.2 BGTFeeDeployer.sol

##### Function: `deployPOLModules(address owner)`

**이 함수는 PoL 관련 핵심 모듈(BeraChef, RewardVaultFactory 등)을 배포합니다.**

- **Inputs**
  - `owner`
    - **Control:** onlyOwner 호출자 제어
    - **Constraints:** non-zero address
    - **Impact:** 각 모듈의 초기 소유권

- **Branches and code coverage**
  - **정상 경로**: `new BeraChef()` → `new RewardVaultFactory()` 등 → 각 모듈 `transferOwnership(owner)` → 이벤트 `POLModulesDeployed(owner)` 발행
  - **오류 경로**: `owner == address(0)` 시 revert

- **Negative behavior**
  - 잘못된 owner 전달로 모듈 제어 불능

- **Negative test**
  - `deployPOLModules(address(0))` 호출 시 revert

---
#### 6.3.3 BGTStaker.sol

##### Function: `stake(uint256 amount)`

**이 함수는 사용자가 BGT를 스테이킹하여 PoL 보상을 받을 자격을 얻습니다.**

- **Inputs**
  - `amount`
    - **Control:** 호출자 제어
    - **Constraints:** `amount > 0` && `amount <= balanceOf(msg.sender)`
    - **Impact:** 스테이킹 금액

- **Branches and code coverage**
  - **정상 경로**: `balances[msg.sender] += amount` && `totalStaked += amount` → `transferFrom(msg.sender, this, amount)` → 이벤트 `Staked(msg.sender, amount)`
  - **오류 경로**: invalid amount 시 revert

- **Negative behavior**
  - 가스 소진을 유발하는 과도한 small-stake 요청

- **Negative test**
  - `stake(0)` 호출 시 revert


##### Function: `claimReward(address to)`

**이 함수는 사용자가 누적된 PoL 보상을 인출합니다.**

- **Inputs**
  - `to`
    - **Control:** 호출자 제어
    - **Constraints:** non-zero address
    - **Impact:** 보상 수령자

- **Branches and code coverage**
  - **정상 경로**: `pending = earned(msg.sender)` → `require(pending > 0)` → `rewardToken.transfer(to, pending)` → 이벤트 `RewardClaimed(msg.sender, to, pending)`
  - **오류 경로**: `pending == 0` 또는 `to == address(0)` 시 revert

- **Negative behavior**
  - 재진입 공격 시 중복 보상 인출 시도

- **Negative test**
  - `claimReward(address(0))` 또는 `claimReward()` where no rewards → revert


##### Function: `recoverERC20(address tokenAddress, uint256 tokenAmount)`

**이 함수는 잘못 전송된 ERC20 토큰을 회수합니다.**

- **Inputs**
  - `tokenAddress`
    - **Control:** onlyOwner
    - **Constraints:** `tokenAddress != rewardToken` (보상 토큰 회수 금지)
    - **Impact:** 회수 대상 ERC20
  - `tokenAmount`
    - **Control:** onlyOwner
    - **Constraints:** `tokenAmount > 0`
    - **Impact:** 회수할 토큰 양

- **Branches and code coverage**
  - **정상 경로**: `require(tokenAddress != rewardToken)` → `IERC20(tokenAddress).transfer(owner, tokenAmount)` → 이벤트 `Recovered(tokenAddress, tokenAmount)`
  - **오류 경로**: `tokenAddress == rewardToken` 또는 `tokenAmount == 0` 시 revert

- **Negative behavior**
  - 의도치 않은 토큰 회수로 사용자 자금 손실

- **Negative test**
  - `recoverERC20(rewardToken, X)` 호출 시 revert

---

### 6.4 FeeCollector.sol

#### Function: `collectFee(address token, uint256 amount)`

**이 함수는 외부로부터 수수료 토큰을 수집해 저장합니다.**

- **Inputs**
  - `token`
    - **Control:** onlyFeeCollector 또는 allowList에 등록된 콜러
    - **Constraints:** whitelisted token
    - **Impact:** 저장할 토큰 종류
  - `amount`
    - **Control:** same as above
    - **Constraints:** `amount > 0`
    - **Impact:** 저장할 수수료 양

- **Branches and code coverage**
  - **정상 경로**: 권한 및 토큰 체크 → `IERC20(token).transferFrom(caller, this, amount)` → `feeBalances[token] += amount` → 이벤트 `FeeCollected(token, amount)`
  - **오류 경로**: invalid token 또는 `amount == 0` 시 revert

- **Negative behavior**
  - 토큰 스풀링 공격으로 잘못된 수집 유도

- **Negative test**
  - `collectFee(unlistedToken, 100)` 호출 시 revert

---
#### Function: `claimFees(address token)`

**이 함수는 누적된 수수료를 RewardVault로 분배합니다.**

- **Inputs**
  - `token`
    - **Control:** 호출자 제어
    - **Constraints:** whitelisted token
    - **Impact:** 분배 대상 토큰

- **Branches and code coverage**
  - **정상 경로**: `total = feeBalances[token]` → `require(total > 0)` → loop over `vaults` → `safeTransfer(vault, share)` → 이벤트 `FeesClaimed(token, vault, share)`
  - **오류 경로**: `total == 0` → `revert NothingToClaim()`; unsupported token → `revert UnsupportedToken()`

- **Negative behavior**
  - front‑running으로 mempool에서 선제적 수령

- **Negative test**
  - `claimFees(validToken)` with zero balance → revert

---
### 6.5 POLDDeployer.sol

#### Function: `deployPOLModules(address owner)`

**이 함수는 Proof‑of‑Liquidity 핵심 모듈들을 배포하고 초기화합니다.**

- **Inputs**
  - `owner`
    - **Control:** onlyOwner
    - **Constraints:** non-zero address
    - **Impact:** 모듈 권한 위임 대상

- **Branches and code coverage**
  - **정상 경로**: 각 PoL 모듈(BeraChef, Distributor, etc.) `new` 인스턴스 생성 → `transferOwnership(owner)` 호출 → 이벤트 `POLCoreDeployed(module, owner)` 반복 발행
  - **오류 경로**: `owner == address(0)` 시 revert

- **Negative behavior**
  - 잘못된 owner 전달로 모듈 초기 패치 실패

- **Negative test**
  - `deployPOLModules(address(0))` 호출 시 revert

---

## 7. 기타 (v0_, v1_, v2_, WBERA 등)

### 7.1 WBERA.sol

#### Function: `deposit() payable`

**이 함수는 사용자가 Native BERA(ETH 유사)를 지불해 WBERA 토큰을 민트합니다.**

- **Inputs**
  - 없음 (전송된 `msg.value`가 wrapping 금액으로 사용됨)
    - **Control:** 호출자 제어
    - **Constraints:** `msg.value > 0`
    - **Impact:** 민트될 WBERA 양 결정 (`amount = msg.value`)

- **Branches and code coverage**
  - **정상 경로**: `require(msg.value > 0, "ZeroDeposit")` → `_mint(msg.sender, msg.value)` → 이벤트 `Deposit(msg.sender, msg.value)` 발행
  - **오류 경로**: `msg.value == 0` 시 `revert ZeroDeposit()`

- **Negative behavior**
  - 중복 호출 혹은 잘못된 fallback 처리로 wrapped 금액 불일치 유발
  - 가스 소비량을 조작해 DoS 유발 가능(반복 작은 deposit)

- **Negative test**
  - `deposit()` 호출 시 `msg.value == 0`이면 반드시 `ZeroDeposit()`로 revert

---
#### Function: `withdraw(uint256 amount)`

**이 함수는 사용자가 보유한 WBERA를 소각하고, 동일량의 Native BERA를 회수합니다.**

- **Inputs**
  - `amount`
    - **Control:** 호출자 제어
    - **Constraints:** `amount > 0` && `balanceOf(msg.sender) >= amount`
    - **Impact:** 반환될 Native BERA 양

- **Branches and code coverage**
  - **정상 경로**: `require(amount > 0, "ZeroWithdraw")` && `require(balance >= amount, "InsufficientBalance")` → `_burn(msg.sender, amount)` → `(bool success, ) = msg.sender.call{value: amount}(""); require(success, "ETHTransferFailed")` → 이벤트 `Withdraw(msg.sender, amount)` 발행
  - **오류 경로**: `amount == 0` → `revert ZeroWithdraw()`; `balance < amount` → `revert InsufficientBalance()`; ETH 전송 실패 시 `revert ETHTransferFailed()`

- **Negative behavior**
  - 재진입 공격: Native BERA 전송 방식(`call`)에 대한 reentrancy guard 부재 시
  - gas stipend 문제로 ETH 전송 실패 가능성

- **Negative test**
  - `withdraw(0)` 호출 시 반드시 `ZeroWithdraw()`로 revert
  - `withdraw(exceedBalance)` 호출 시 반드시 `InsufficientBalance()`로 revert
  - ETH 수신 실패 시 `ETHTransferFailed()` 발생 검증

> **Note:** 각 섹션의 세부 위협 모델은 동일한 양식(`Inputs` / `Branches…` / `Negative behavior` / `Negative test`)으로 작성되어야 합니다. 이후 각 파일별로 상세 내용을 순차적으로 추가해 드릴 수 있습니다.

