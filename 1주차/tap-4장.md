# 4장: 모의 객체를 사용한 상호 작용 테스트
작업 단위가 서드 파티 함수와 올바르게 상호 작용하는지 어떻게 테스할까? 답은 목(Mock)!
![image](https://github.com/user-attachments/assets/de7a5059-ea52-4d9b-8868-f78dfcd48178)

- **상호 작용 테스트**
	- 작업 단위가 제어할 수 없는 영역에 있는 의존성과 어떻게 상호작용하고 메시지(함수)를 보내는지 확인하는 방법
	- 모의 함수/객체(mock function/objects)를 사용하여 외부 의존성 호출을 검증
	- 서드 파티 모듈 및 객체, 시스템 등을 의미하는 세 번째 종료점과 관련(첫 번째: '반환 값', 두 번째: '상태 값 변경')
- 목(Mock)
	- 단위 테스트에서의 **종료점**
	- 어떤 대상을 흉내 내어 만들어서 **호출 여부를 검증**하는 것이 중요
- 스텁(Stub)
	- 검증할 필요는 없고 하나의 테스트에 여러개를 사용하면 된다.
	- 데이터나 동작이 **작업 단위로 들어오는 것**을 나타내고 **종료점은 나타내지 않는다**.
	- 상호 작용이 발생하는 지점으로 볼 수 있으나, 작업 단위의 최종 결과를 나타내지 않는다.

### 함수 지향적
1. 표준 방식 - 매개변수를 추가  주입
```ts
// 가장 일반적인 방식으로, 의존 객체를 함수나 클래스의 매개변수로 주입
function process(logger: { log: (msg: string) => void }) {
  logger.log('처리 중입니다');
}

// 목 객체 생성
const mockLogger = { log: jest.fn() };

process(mockLogger);

// 상호작용 검증
// log()가 올바른 인자로 호출되었는가?
expect(mockLogger.log).toHaveBeenCalledWith('처리 중입니다');
```
2. 함수형 방식 - 부분 적용(커링) 또는 팩토리 함수
```ts
// 팩토리 함수 또는 부분 적용(커링)을 통해 의존성을 주입  
// 순수 함수 스타일로 테스트하기 유리
function createProcessor(logger: { log: (msg: string) => void }) {
  return function process() {
    logger.log('처리 중입니다');
  };
}

const mockLogger = { log: jest.fn() };

const processor = createProcessor(mockLogger);
processor();

// Mock을 커링의 첫 번째 인자로 주입하여 테스트가 쉬운 구조로 분리
// log()가 올바른 인자로 호출되었는가?
expect(mockLogger.log).toHaveBeenCalledWith('처리 중입니다');
```
3. 모듈 방식 - 모듈 의존성을 추상화하여 주입
```ts
// 의존하는 모듈을 직접 import하지 않고, 외부에서 주입하여 테스트 시 대체 가능
// processor.js
export function createProcessor({ logger }) {
  return {
    run() {
      logger.log('모듈 처리 실행');
    },
  };
}

// 테스트
const mockLogger = { log: jest.fn() };

const processor = createProcessor({ logger: mockLogger });
processor.run();

// 실제 모듈을 목으로 바꾸기 쉬움 → 테스트에서 유리함
// log()가 run()을 통해 올바른 인자로 호출되었는가?
expect(mockLogger.log).toHaveBeenCalledWith('모듈 처리 실행');
```

### 객체 지향적
1. 객체 지향 방식 – 타입이 지정되지 않은 객체 또는 인터페이스 기반 객체 사용
```ts
// ts 타입 (인터페이스 명시)
interface Logger {
  log(msg: string): void;
}

class Service {
  constructor(private logger: Logger) {}

  execute() {
    this.logger.log('서비스 실행');
  }
}

const mockLogger: Logger = { log: jest.fn() };

const service = new Service(mockLogger);
service.execute();

// 명시적 인터페이스 주입으로 테스트 가능
// log()가 올바른 인자로 호출되었는가?
expect(mockLogger.log).toHaveBeenCalledWith('서비스 실행');
```
2. 부분 모의 객체 방식 – 실제 클래스를 상속 받아 일부만 대체
```ts
// 실제 클래스를 기반으로, 일부 메서드만 override하여 가짜 동작 구현
// 통합 테스트나 특정 조건 검증에 유리
class PaymentService {
  send(amount: number): string {
    // 실제 결제 로직
    return `결제 완료: ${amount}`;
  }
}

class MockPaymentService extends PaymentService {
  send = jest.fn().mockReturnValue('결제 MOCK');
}

function processPayment(service: PaymentService) {
  return service.send(10000);
}

const mockService = new MockPaymentService();
const result = processPayment(mockService);

// 실제 클래스의 구조와 흐름을 그대로 유지하면서, 부분만 mocking
// send()가 올바른 인자로 호출되었는가?
expect(mockService.send).toHaveBeenCalledWith(10000);
expect(result).toBe('결제 MOCK');
```

### 복잡한 인터페이스 다루기
- 인터페이스 주입(다형성)
- 복잡한 인터페이스를 추상화하여 구체적인 구현을 직접 참조하지 않기
- ISP 활용 > 더 작은 인터페이스, 함수를 더 적게, 이름을 더 명확하게, 매개변수를 덜 사용하게

1. 인터페이스 명확화 및 ISP 적용
```ts
// 역할(인터페이스)에 의존하고, 구현(클래스)에 의존하지 않도록 구성
// 더 작고 명확한 인터페이스로 분리 (ISP 적용)
interface Loggable {
  log(message: string): void; // 이름 명확, 매개변수 최소화
}

interface PaymentSender {
  send(amount: number): string;
}

```
2. 서비스 클래스에 인터페이스 주입
```ts
// 역할(인터페이스)에 의존하고, 구현(클래스)에 의존하지 않도록 구성
// 인터페이스 기반으로 의존성 주입 (DI + 다형성)
class Service {
  constructor(private logger: Loggable) {}

  execute() {
    this.logger.log('서비스 실행');
  }
}

```
3. 실제 구현과 테스트용 구현 분리
```ts
// 실제 구현체
class ConsoleLogger implements Loggable {
  log(message: string) {
    console.log(`[LOG]: ${message}`);
  }
}

// 테스트용 Mock
const mockLogger: Loggable = {
  log: jest.fn(),
};

// 사용
const service = new Service(mockLogger);
service.execute();

expect(mockLogger.log).toHaveBeenCalledWith('서비스 실행');

```
4. PaymentService도 인터페이스로 추상화
```ts
// 기존 PaymentService 구현체
class PaymentService {
  send(amount: number): string {
    return `결제 완료: ${amount}`;
  }
}

// PaymentSender 인터페이스로 구현 숨기기
// PaymentService → PaymentSender 로 작고 명확한 역할 분리
class RealPaymentService implements PaymentSender {
  send(amount: number): string {
    return `결제 완료: ${amount}`;
  }
}

function processPayment(service: PaymentSender) {
  return service.send(10000);
}

// 테스트: 인터페이스 기반 Stub/Mock
const mockPaymentService: PaymentSender = {
  send: jest.fn().mockReturnValue('결제 MOCK'),
};

const result = processPayment(mockPaymentService);

expect(mockPaymentService.send).toHaveBeenCalledWith(10000);
expect(result).toBe('결제 MOCK');
```

## 정리
- **상호 작용 테스트**는 Mock을 사용해 외부 의존성과의 **호출 여부와 인자**를 검증한다.
- **IoC/DI**를 통해 의존성을 외부에서 주입하고, 테스트 가능한 구조로 만든다.
- **ISP**를 적용하면 더 작고 명확한 **인터페이스로 역할을 분리**할 수 있다.
- **Stub**은 값 제어용, **Mock**은 행동 검증용이며 용도와 종료점이 다르다.
