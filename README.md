# 대용량 정형 데이터에 대한 MapReduce 기반 분산 가명 처리 시스템 

<br/>
<br/>
<br/>

## 목차
0. 프로젝트 성과
1. 담당 업무 및 나의 기여도
2. 프로젝트 개요  
   2.1 프로젝트 개요  
   2.2 프로젝트 소개  
3. 제안 시스템  
   3.1 사용자 입력 파일  
   3.2 제안 시스템 흐름도  
4. 실험  
   4.1 실험 환경  
   4.2 실험 데이터  
   4.3 파라미터 최적화  
   4.4 실험 결과
   
   
<br/>
<br/>
<br/>

# 0. 프로젝트 성과 
1. <code style="color : red">**[과제 목표 달성]** 2TB 개인 정보 가상 데이터 8가지 기능 1시간 수행 성공 </code> 
2. <code style="color : red">**[S/W 저작권 등록]** 대용량 파일에 대한 MapReduce 기반 분산 가명처리 시스템</code>
3. <code style="color : red">**[KSC2023]** 대규모 정형 데이터를 위한 맵리듀스 기반 고속 가명 처리 시스템 개발</code>

<br/>
<br/>
<br/>

# 1. 담당 업무 및 나의 기여도 
0. 나의 기여도 : **100%**
1. **클러스터 환경 구축**
   - 16개 노드로 이루어진 Hadoop 클러스터를 직접 구축
   - https://joyous-shell-10e.notion.site/d81af1f1be5f4f5ca969f97b1fff8063?pvs=4
   
2. **MapReduce 프로그램 코드 최적화**

|||
|:---:|---|
|문제1|리듀서 함수 내 데이터의 모든 컬럼(N개)과 입력된 가명 처리 기능(M개)의 수 만큼 반복되는 비효율적인 이중 for-loop구조 발견|
|해결1|가명 처리 필요한 컬럼 위치 사전에 식별 및 저장하여 단일 반복 구조로 개선 **계산 복잡도 O(N*M)에서 O(M) 단축** |
|문제2|맵과 리듀서 사이의 데이터 전송 병목 현상 발생|
|해결2|컴바이너(Combiner)를 통해 합계, 최소값 등의 통계값을 로컬에서 미리 집계(Aggregation)하여 리듀서로의 **데이터 전송량 감소**|
 
3. **MapReduce 프로그램 파라미터 최적화**  
   - 하둡 공식 문서 <a href ="https://hadoop.apache.org/docs/r3.0.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml">(링크)</a> 와 관련 서적 탐구하며 다양한 설정값에 대한 실험 진행 

<br/>
<br/>
<br/>

# 2. 프로젝트 개요
## 2.1 프로젝트 개요

|||
|:---:|--|
|**과제명**|대용량 정형 데이터 대상 개인정보 가명,익명화를 위한 자동처리 기술|
|**지원 기관**|정보통신기획평가원 (IITP)|
|**프로젝트 기간**| 2023.03 - 2023.12 |
|**연구 목표**| 2TB 개인정보 파일을 4시간 내에 가명/익명처리 완료 <br/> * 기준: 16 노드 클러스터|

<br/>
<br/>
<br/>

## 2.2 프로젝트 소개
- TB급 대용량 데이터에 대한 최초의 Hadoop MapReduce 기반 분산 가명처리 시스템 개발
- 총 8개의 대표적인 가명처리 기능 제공
  
|기능|설명|
|:---:|---|
|상하단코딩 <br> (Top and Bottom Coding)|적은 수의 분포를 가진 양 끝단의 정보를 평균값으로 대체하여 식별성을 낮추는 기법|
|부분 총계 <br> (Micro-Aggregation)|특정 그룹 내, 다른 정보에 비하여 오차 범위가 큰 항목을 평균값 등으로 대체|
|난수화 <br> (Randomization)|주어진 입력 값에 대해 예측이 불가능하고 패턴이 없는 값으로 임의 할당|
|열 삭제 <br> (Column Deletion)|개인정보 열 전체 삭제|
|마스킹 <br> (Masking)|특정 항목의 일부 또는 전부를 공백 또는 문자로 대체|
|부분 삭제 <br> (Partial Deletion)|개인정보 전체 삭제 대신 일부 삭제|
|라운딩 <br> (Rounding)|올림, 내림, 반올림 등의 기준을 적용하여 집계 처리|
|암호화 <br> (Encryption)|지정된 열을 SHA-256 알고리즘을 사용하여 암호화|

<br/>
<br/>
<br/>

# 3. 제안 시스템


## 3.1  사용자 입력 파일
<img src="/1. Image/Json_format.png" title="사용자 입력 파일 형태"></img><br/>  
- 사용자가 입력한 Json 파일을 통해 가명 처리 시스템 수행
- {Key:Value = 가명 처리 수행할 컬럼명 : ["수행 기능", "옵션"]} 형식으로 입력
  
<br/>
<br/>
<br/>

## 3.2 제안 시스템 전반적인 흐름도 
<img src="/1. Image/Overview.png" title="프로그램 흐름도"></img><br/>

- **Job1** : 가명화 연산 수행을 위해 통계 정보 획득 단계
  - Map, Combiner, Reduce 함수로 구성
  - 수행 기능 : 상하단 코딩, 부분 총계 [평균, 표준편차] / 난수화 [최솟값, 최댓값]
  - **Map()** : 입력 데이터에서 특정 조건에 맞는 데이터 필터링
  - **Combiner()** : Map의 출력을 합계, 제곱합, 개수, 최솟값, 최댓값 등으로 집계하여 전송
  - **Reduce()** : Combiner의 출력을 통해 열의 평균/표준편차, 최솟값/최댓값 계산
  
- **Job2** : 8가지 가명 처리를 실질적으로 수행하는 작업
  - 불필요한 Reduce 단계를 생략한 Map-Only 작업
  - **Map()** : 레코드 별로 한 줄씩 읽어 가명 처리에 해당하는 열일 경우 연산 수행
  - Map 함수의 결과값은 최종적으로 가명 처리가 완료된 데이터로 HDFS에 저장
 
<br/>
<br/>
<br/>

# 4. 실험

## 4.1 실험 환경  
 
<table>
  <tr>
    <td rowspan="2">노드 수</td>
    <td>네임 노드 : 1대</td>
  </tr>
  <tr>
  <td>데이터 노드 : 15대</td>
  </tr>
  <tr>
    <td rowspan="3">노드 성능</td>
    <td>CPU : Intel(R) Xeon(R) Silver@2.10GHz</td>
  </tr>
  <tr>
    <td>Memory : 64GB</td>
  </tr>
  <tr>
    <td> SSD : 500GB </td>
  </tr>
  <tr>
    <td> 운영체제 </td>
    <td> Linux </td>
  </tr>
  <tr>
    <td> Hadoop </td>
    <td> Hadoop-3.3.5 </td>
  </tr>

</table>

<br/>
<br/>
<br/>

## 4.2 실험 데이터
- (주)이지서티에서 제공한 주민등록번호, 성별, 혈액형, 주거래은행, 근무년수, 연봉 등으로 구성된 가상 개인 정보를 사용
- 수치형 8개, 문자형 22개로 총 30개의 열로 구성된 정형 데이터
- 500GB / 1000GB / 1500GB / 2000GB 대규모 데이터로 실험 진행
- 2000GB 기준 약 60억 행으로 구성  
<img src="/1. Image/Data.png" title="데이터 일부"></img><br/>

<br/>
<br/>
<br/>

## 4.3 파라미터 최적화 
- 하둡은 파라미터에 따라 수행 시간과 속도가 크게 영향 받음
- 최적의 파라미터는 그리드 서치 (Grid Search)방식으로 모니터링과 실험을 거쳐 최적화된 값 확인
  
|매개변수|설명|서치 범위|
|:---:|:---:|:---:|
|dfs.block.size|HDFS에서 파일을 블록으로 나눌 때 각 블록의 크기|{128MB, 256MB, 512MB}|
|mapreduce.job.maps|Map 단계에서 실행할 맵 태스크 개수|{10, 30, 50, 100, 200}|
|mapreduce.job.reduces|Reduce 단계에서 실행할 리듀스 태스크 개수|{10, 30, 50, 100, 200}|
|mapreduce.input.fileinputformat.split.minsize|하나의 입력 파일이 분할 가능한 최소 크기|{128MB, 256MB, 512MB}|
|mapreduce.input.fileinputformat.split.maxsize|하나의 입력 파일이 분할 가능한 최대 크기|{128MB, 256MB, 512MB}|
|mapreduce.reduce.memory.mb|MapReduce 작업의 Reduce 태스크가 사용할 수 있는 메모리 양|{2GB, 4GB, 8GB, 16GB}|
|mapreduce.map.memory.mb|MapReduce 작업의 Map 태스크가 사용할 수 있는 메모리 양|{2GB, 4GB, 8GB, 16GB}|
|mapred.compress.map.output|Map 단계 출력의 압축 여부|{True}|
|mapred.output.compression.type|출력 데이터의 압축 유형 결정|{BLOCK}|
|mapred.map.output.compression.codec|맵 작업의 출력 데이터에 사용할 압축 코덱|{Snappy}|

<br/>
<br/>
<br/>

## 4.4 실험 결과
<img src="/1. Image/Result.png" title="데이터 일부"></img><br/>

- 본 연구는 정보화 시대의 데이터 활용성을 위한 **대용량 데이터에 대한 맵리듀스 기반 가명 처리 시스템**을 제안한다.
- 총 8개의 대표적인 가명 처리 기능 동시 수행으로 실험
- HDFS에 있는 데이터를 읽어 분할하고 Job1과 Job2를 거쳐 가명 처리를 수행하여 저장하는 데 2TB 데이터 기준 61.83분이 소요됨을 확인
- 제안하는 시스템이 실제 산업에서도 대규모 정형 데이터 가명 처리에 실용적으로 적용할 수 있음을 보여준다.

<br/>
<br/>
<br/>



