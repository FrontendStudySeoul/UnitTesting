# 3장: 의존성 분리와 스텁

## 의존성 유형
![image](https://github.com/user-attachments/assets/1f785dd6-4f08-4fac-9bcc-fa736eedf846)

- **외부로 나가는** 의존성
	- **작업 단위의 종료점**을 나타내는 의존성. 작업 단위 내의 **특정 논리 흐름의 끝**
	- ex) logger 함수 호출, DB 저장, 메일 발송, API or webhook 노티 등
- **내부로 들어오는** 의존성
	- **종료점을 나타내지 않은** 의존성. 내부에서 간접적으로 들어오는 입력 또는 동작
	- ex) DB 쿼리 결과, 파일 시스템의 파일 내용, 네트워크 응답 결과 등

## 스텁(Stub)과 목(Mock)
![image](https://github.com/user-attachments/assets/2e7e0249-0ac4-45fc-bb57-1fdd6fb9afc0)

- SUT(System Under Test)
	- 현재 테스트 대상
	- 테스트에서 **검증의 주체**가 되는 클래스, 함수, 모듈
- 스텁/더미(Stub/Dummy)
	- 내부로 들어오는 의존성(간접 입력)을 끊어줌
	- 더미, 가짜 모듈, 객체, 동작, 데이터를 **코드 내부**로 보내는 함수
	- 검증하지 않는 데이터
	- 하나의 테스트에 여러 스텁 선언 가능
	- *목적: 테스트 대상 코드가 외부 시스템이나 데이터에 의존하지 않게 동작 가능*
- 목(mock)
	- 외부로 나가는 의존성(간접 출력 또는 종료점)을 끊어줌
	- 가짜 모듈, 객체, 호출 여부를 검증하는 함수
	- 단위 테스트에서 종료점을 나타냄
	- 테스트와 목은 1:1 관계
	- *목적: 간접 출력으로 독립적으로 로직 검증이 가능*
- 페이크/테스트 더블(fake)
	- 스텁과 목을 포함.(스텁 하나만으로 간주하진 않음)
	- *목적: 실제 구현을 대체하는 가벼운 구성 요소를 제공하여 테스트*

| 구분        | 스텁 (Stub)                    | 목 (Mock)                     |
| --------- | ---------------------------- | ---------------------------- |
| **목적**    | 정해진 값을 반환하게 함 (제어용)          | 호출 여부, 횟수, 인자 등을 검증 (검증용)    |
| **초점**    | 리턴 값                         | 행동, 상황(호출 여부)                |
| **사용 시점** | 테스트 대상이 의존하는 객체의 응답을 바꿔야 할 때 | 함수가 호출되었는지를 검증해야 할 때         |
| **예**     | "항상 success를 리턴해줘"           | "sendEmail이 정확히 한 번 호출되었는가?" |

### Stub 예제: 리턴 값만 제어
```js
// EmailService.js
export class EmailService {
  sendEmail(email: string, content: string) {
    return 'success';
  }
}

// UserService.js
export class UserService {
  constructor(private emailService: EmailService) {}

  registerUser(email: string) {
    const result = this.emailService.sendEmail(email, 'Welcome!');
    return result === 'success';
  }
}

// test/UserService.test.ts
test('registerUser returns true when email sending succeeds', () => {
  const stubEmailService = {
    sendEmail: jest.fn().mockReturnValue('success'), // ✅ Stub: 리턴 값만 고정
  };

  const userService = new UserService(stubEmailService); // SUT
  expect(userService.registerUser('test@example.com')).toBe(true);
});
```

### Mock 예제: 호출 여부 검증
```js
test('registerUser sends a welcome email', () => {
  const mockSendEmail = jest.fn().mockReturnValue('success'); // ✅ Mock: 행동을 감시
  const mockEmailService = { sendEmail: mockSendEmail };

  const userService = new UserService(mockEmailService); // SUT
  userService.registerUser('test@example.com');

  // ✅ 호출 여부와 인자를 검증
  expect(mockSendEmail).toHaveBeenCalledTimes(1);
  expect(mockSendEmail).toHaveBeenCalledWith('test@example.com', 'Welcome!');
});

```

## 스텁 사용과 이유
테스트는 언제 실행하던 **이전 실행과 같은 결과를 보장해야한다** 테스트를 구성하는 여러 변수, 환경을 통제할 수 없으면 테스트는 불완전해진다.
ex) e2e 테스트시 네트워크 이슈, DB 연결 이슈, 여러 서버 이슈 등

> 많은 경우에 "그냥 다시 실행", "새로 고침", "간헐적 테스트 실패" 등으로 테스트 실패를 무시함...
> 
> Q. 다른 분들은 어떻게 현 상황을 대응하는지?

### 스텁 사용시 일반적인 설계 방식
- 함수
	- 함수를 매개변수로
	- 부분 적용(커링)
	- 팩토리(고차) 함수
	- 생성자 함수
- 모듈
	- 모듈 주입
- 객체 지향
	- 클래스 생성자 주입
	- 객체를 매개변수화(덕 타이핑)
	- 공통 인터페이스를 매개변수(타입 스크립트)

위와 같은 방식을 SOLID 원칙에서의 **의존성 역전(DI: Dependancy Inversion)**, **제어의 역전(IoC: Inversion of Control)** 방식이다. (**DI(Dependency Injection)** 는 DIP를 구현하는 **전략**)

- **IoC (Inversion of Control: 제어의 역전)**
	- **정의**: 프로그램의 흐름 제어(객체 생성, 실행 등)를 개발자가 아닌 **외부(호출자나 프레임워크)** 가 담당하는 방식
	- **효과**: 모듈 간 결합도 낮춤, 유연성 증가
- **DIP (Dependency Inversion Principle: 의존성 역전 원칙)**
	- **정의**: 상위 모듈이 하위 모듈의 구현 세부사항에 의존하지 않도록 하고, **추상화(인터페이스)** 에 의존하게 만드는 원칙
	- **DI (Dependency Injection)** 는 DIP를 실현하는 **전략/기술**
- 심(Seam)
	- **정의**: 대체 가능한 지점, 외부 동작을 바꿀 수 있는 **지점**

| 설계 방식               | 설명                    | 관련 원칙 | 예시                            | 어떤 "역전"?      |
| ------------------- | --------------------- | ----- | ----------------------------- | ------------- |
| **함수 매개변수 주입**      | 외부에서 함수를 주입받음         | IoC   | `fn(callback)`                | 흐름 제어의 역전     |
| **부분 적용 (커링)**      | 의존성 일부를 나중에 주입        | IoC   | `fn(a)(b)`                    | 순차적 주입을 통한 제어 |
| **팩토리 함수**          | 의존성을 인자로 받아 객체 생성     | DIP   | `createService(dep)`          | 의존성 주입        |
| **생성자 함수**          | 생성 시 외부 의존 주입         | DIP   | `new A(dep)`                  | 구현 분리         |
| **모듈 주입**           | 모듈을 직접 import하지 않고 주입 | DIP   | `init({moduleA})`             | 정적 의존성 제거     |
| **클래스 생성자 + 인터페이스** | 타입 기반으로 의존성 주입        | DIP   | `constructor(logger: Logger)` | 추상화 의존        |
| **덕 타이핑 객체 주입**     | 구조만 맞추면 되는 유연한 설계     | DIP   | `fn({ log: () => {} })`       | 구조적 추상화       |
| **공통 인터페이스 주입**     | TS interface를 이용한 추상화 | DIP   | `logger: ILogger`             | 구현체 독립화       |
### 함수 지향적
1. **함수 매개변수 주입 (IoC)**
```ts
function runTask(logger: (msg: string) => void) {
  // 심: 외부에서 logger를 바꿀 수 있음
  logger('작업 시작');
}
runTask((msg) => console.log(msg)); // 외부에서 제어
runTask(jest.fn()) // 테스트 시 Stub 또는 Mock
```
2. 모듈 주입(DIP) + 팩토리 함수 방식
```ts
function createUserService({ emailService }) {
  return {
    register(email) {
      // 심: emailService를 외부에서 제어 가능
      emailService.send(email);
    },
  };
}

const stubEmailService = {
  send: jest.fn(), // Mock: 호출 검증 가능 / Stub: 동작 고정 가능
};

// emailService를 외부 모듈로 주입
const userService = createUserService({ emailService: stubEmailService });
```
3. 커링(부분 적용) (IoC)
```ts
function createLogger(prefix: string) {
  return function log(message: string) {
    console.log(`[${prefix}] ${message}`);
  };
}


const debugLog = createLogger('DEBUG'); // prefix를 먼저 고정
debugLog('App started'); // 나중에 message 주입

// Stub: createLogger('MOCK')으로 가짜 로거 주입 가능
const mockLog = jest.fn();
const mockLogger = createLogger('MOCK');
```
4. 덕 타이핑 (DIP)
```ts
// logger를 요구하는 함수
// logger를 외부로 위임하여 IoC 효과
// log 타입의 구조에만 의존 (DIP). 구현체 X
function processTask(logger: { log: (msg: string) => void }) { 
  logger.log('Task started');
}

// 테스트용 stub logger
const stubLogger = {
  log: jest.fn(), // 형태만 맞으면 됨. Mock: 호출 검증 가능 / Stub: 응답 제어도 가능
};

processTask(stubLogger);
```

### 객체 지향적
1. 클래스 생성자 + 인터페이스 주입 (DIP)
```ts
// 의존 객체를 '생성자 매개변수'로 전달
// 가장 많이 쓰이는 방법이며, '불변성을 보장'하고 '명확한 의존성 표현'이 가능
interface Logger {
  log(msg: string): void;
}

class Service {
  constructor(private logger: Logger) {
	  // 심: 외부에서 logger 구현체를 바꿀 수 있음
  }

  doWork() {
    this.logger.log('일 시작!');
  }
}

class ConsoleLogger implements Logger {
  log(msg: string) {
    console.log(msg);
  }
}

const service = new Service(new ConsoleLogger()); // 추상화에 의존

const fakeLogger = {
  log: jest.fn(), // Mock: 호출 검증 가능 / Stub: 동작 고정 가능
};

const service = new Service(fakeLogger); // 테스트용 logger 주입
```
2. Setter 주입
```ts
class Service {
  private logger: Logger;

  setLogger(logger: Logger) {
    this.logger = logger;
  }

  execute() {
    this.logger.log('작업 실행');
  }
}

const service = new Service();
service.setLogger({ 
	log: console.log // Mock 또는 Stub
}); // 외부에서 주입. 의존 객체를 setter 메서드로 주입
```
3. 덕 타이핑 주입
```ts
class Service {
  constructor(private logger: { log: (msg: string) => void }) {}

  execute() {
    this.logger.log('실행됨');
  }
}

const service = new Service({
  log: (msg) => console.log('[LOG]', msg),
});
const mockService = new Service({
  log: jest.fn(), // ✅ Mock: 호출 여부 검증 / Stub: 동작 고정 가능
});

```

### 정리
- **IoC**: "제어를 외부로 넘겨라" → 흐름 제어를 외부에서 하도록 설계
- **DIP**: "구현이 아닌 추상화에 의존하라" → 결합도 낮고 테스트 가능한 구조
- **Stub 사용 시**: 위 원칙들이 잘 적용되어 있으면 테스트가 쉽고 유연한 구조가 된다
