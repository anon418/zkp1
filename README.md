# ZKP Example: Multiplier Circuit with Circom & SnarkJS

이 저장소는 **영지식 증명(Zero-Knowledge Proof)**을 학습하기 위한 프로젝트입니다. Circom과 SnarkJS를 사용하여 간단한 곱셈 회로(`c <== a * b`)를 구현하고, 특정 숫자의 두 인수를 알고 있음을 인수 값을 공개하지 않고 증명하는 전 과정을 다룹니다.

---

## 💻 환경 설정 (Prerequisites)

이 프로젝트를 로컬에서 실행하기 위해서는 아래 도구들이 설치되어 있어야 합니다.

1.  **Node.js & npm**: SnarkJS는 JavaScript 라이브러리이므로 Node.js 런타임이 필요합니다.
2.  **Rust**: Circom 컴파일러는 Rust로 작성되었습니다. `rustup`을 통해 설치하는 것을 권장합니다.
    ```bash
    curl --proto '=https' --tlsv1.2 -sSf [https://sh.rustup.rs](https://sh.rustup.rs) | sh
    ```
3.  **Circom**: 아래 명령어로 설치합니다.
    ```bash
    git clone [https://github.com/iden3/circom.git](https://github.com/iden3/circom.git)
    cd circom
    cargo build --release
    cargo install --path circom
    ```
4.  **SnarkJS**: npm을 통해 전역으로 설치합니다.
    ```bash
    npm install -g snarkjs
    ```

---

## 🚀 실행 순서

### 1. 회로 정의

증명하려는 규칙을 Circom 언어로 작성합니다. 이 예제에서는 `a`와 `b`라는 두 개의 비공개(private) 입력값을 곱하면 `c`라는 공개(public) 출력값이 나온다는 것을 증명합니다.

**`example.circom`**
```javascript
pragma circom 2.0.0;

template Multiplier() {
    // Private inputs
    signal input a;
    signal input b;

    // Public output
    signal output c;

    // Constraint
    c <== a * b;
}

component main {public [c]} = Multiplier();
```

**`input.json`** (비공개 입력값)
```json
{
  "a": "3",
  "b": "11"
}
```

### 2. 회로 컴파일

`circom` 명령어를 사용해 회로를 컴파일합니다. 이 과정에서 **제약 조건 파일(.r1cs)**과 **계산 프로그램(.wasm)**이 생성됩니다.

```bash
circom example.circom --r1cs --wasm --sym
```

### 3. 신뢰 설정 (Powers of Tau)

증명 시스템의 보안성과 신뢰도를 위해 암호학적 랜덤 값을 생성하는 단계입니다. (Trusted Setup)

```bash
# 1단계: 새로운 powers of tau 세레모니 시작
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v

# 2단계: 랜덤 값 기여
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="First contribution" -v

# 3단계: Phase 2 준비
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v
```

### 4. 키 생성

신뢰 설정 파일과 회로 제약 조건을 이용해 증명 생성에 필요한 **마스터 키(Proving Key, .zkey)**와 증명 검증에 필요한 **검증 키(Verification Key, .json)**를 생성합니다.

```bash
# Zkey (마스터 키) 생성
snarkjs groth16 setup example.r1cs pot12_final.ptau example_0000.zkey

# 검증키(verification_key.json) 내보내기
snarkjs zkey export verificationkey example_0000.zkey verification_key.json
```

### 5. 증명 생성

비공개 입력값(`input.json`)을 회로에 적용하여 계산 과정을 담은 **Witness**를 생성하고, 이를 마스터 키와 결합하여 최종 **증명(Proof)** 파일을 생성합니다.

```bash
# 1단계: Witness 계산
snarkjs wtns calculate example_js/example.wasm input.json witness.wtns

# 2단계: 증명 생성
snarkjs groth16 prove example_0000.zkey witness.wtns proof.json public.json
```
- `proof.json`: `a=3`, `b=11`이라는 비공개 정보를 숨긴 채, 계산이 올바르게 수행되었다는 증거.
- `public.json`: 공개되는 결과값 `c=33`이 담겨 있습니다.

### 6. 증명 검증 ✅

검증자는 **검증 키, 공개 출력값, 증명** 세 가지만으로 증명자가 비공개 입력값을 정말 알고 있는지 확인할 수 있습니다.

```bash
snarkjs groth16 verify verification_key.json public.json proof.json
```
위 명령어를 실행했을 때 터미널에 `OK` 메시지가 출력되면 증명이 성공적으로 검증된 것입니다. 검증자는 `a`와 `b`가 무엇인지 전혀 모르지만, 증명자가 `c=33`을 만드는 두 인수를 알고 있다는 사실을 100% 신뢰할 수 있습니다.

---

## 📄 주요 파일 설명

- **`example.circom`**: ZKP 규칙이 정의된 회로 파일.
- **`input.json`**: 회로에 제공할 비공개 입력값.
- **`example.r1cs`**: 회로의 수학적 제약 조건 파일.
- **`example.wasm`**: Witness 계산을 위한 WebAssembly 프로그램.
- **`.zkey`**: 증명 생성을 위한 마스터 키 (Proving Key).
- **`verification_key.json`**: 증명 검증을 위한 키.
- **`witness.wtns`**: 비공개 입력값과 계산 과정이 포함된 파일.
- **`proof.json`**: 비공개 정보를 숨기고 생성된 영지식 증명.
- **`public.json`**: 외부에 공개되는 출력값.
