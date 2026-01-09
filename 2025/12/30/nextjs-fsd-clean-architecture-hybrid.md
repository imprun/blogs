# Next.js + FSD + Clean Architecture: 하이브리드 아키텍처 설계

> **작성일**: 2025년 12월 30일
> **카테고리**: Frontend Architecture
> **키워드**: Next.js, Feature-Sliced Design, Clean Architecture, FSD, 프론트엔드 아키텍처, 도메인 주도 설계

## 요약

Feature-Sliced Design(FSD)과 Clean Architecture는 서로 다른 방향에서 프론트엔드 아키텍처 문제를 해결한다. FSD는 "UI에서 도메인으로" 하향식 접근을, Clean Architecture는 "도메인에서 UI로" 상향식 접근을 취한다. 두 아키텍처를 결합하면 FSD의 명확한 레이어 구조와 Clean Architecture의 비즈니스 로직 분리라는 장점을 모두 취할 수 있다. 이 글에서는 Next.js 환경에서 두 아키텍처를 효과적으로 결합하는 방법을 다룬다.

## 문제 상황: FSD만으로는 부족한 경우

### FSD의 한계

[Feature-Sliced Design](https://blog.imprun.dev/70)은 프론트엔드 프로젝트 구조를 표준화하는 강력한 방법론이다. 하지만 복잡한 비즈니스 로직을 다룰 때 몇 가지 한계가 드러난다.

**1. Use Case 계층의 부재**

FSD의 features 레이어는 "사용자 상호작용"을 담당한다. 로그인 버튼, 장바구니 추가 같은 UI 액션을 구현한다. 하지만 "주문 생성 시 재고 확인 → 결제 처리 → 알림 발송"과 같은 복잡한 비즈니스 워크플로우를 어디에 배치해야 하는지 명확하지 않다.

```
features/
├── create-order/     # 주문 생성 버튼 UI는 여기
│   └── ui/
└── ??? # 주문 검증 → 재고 확인 → 결제 → 알림 로직은?
```

**2. Repository 패턴의 부재**

FSD에서 API 호출은 각 슬라이스의 `api` 세그먼트에 분산된다. 이는 간단한 CRUD에는 적합하지만, 다음과 같은 상황에서 문제가 된다:

- 동일한 데이터를 여러 곳에서 다른 형태로 조회
- API 응답을 도메인 모델로 변환하는 로직 중복
- 캐싱 전략의 일관성 부재

**3. 의존성 역전 원칙(DIP) 미적용**

FSD는 상위 레이어가 하위 레이어에 직접 의존한다. features가 entities를 직접 import하고, entities가 shared/api를 직접 호출한다. 이는 다음 문제를 야기한다:

- 테스트 시 실제 API 호출 필요
- API 변경 시 여러 레이어 수정 필요
- 프레임워크 교체 어려움

### Clean Architecture가 해결하는 것

Clean Architecture는 "비즈니스 규칙이 프레임워크에 종속되지 않아야 한다"는 원칙을 따른다.

```
호텔로 비유하면:
- 도메인 계층: 호텔의 운영 정책 (체크인 규칙, 요금 정책)
- Use Case 계층: 프론트 데스크 직원 (정책에 따라 업무 수행)
- 인프라 계층: 예약 시스템, 결제 단말기 (교체 가능한 도구)
- UI 계층: 로비, 안내 표지판 (고객 접점)
```

호텔 정책은 예약 시스템이 바뀌어도 변하지 않는다. 마찬가지로 "주문 금액은 1만원 이상이어야 한다"는 비즈니스 규칙은 React가 Vue로 바뀌어도 동일해야 한다.

## FSD vs Clean Architecture: 핵심 차이점

| 관점 | FSD | Clean Architecture |
|------|-----|-------------------|
| **개발 방향** | Pages-first (하향식) | Domain-first (상향식) |
| **레이어 구성** | 7개 (app → shared) | 4개 (Entities → Frameworks) |
| **슬라이싱 기준** | 기능 단위 (user, product) | 기술 역할 (repository, use case) |
| **의존성 규칙** | 하위 레이어만 import 가능 | 안쪽 레이어만 의존 가능 |
| **DIP 적용** | 미적용 (직접 의존) | 필수 (인터페이스 분리) |
| **테스트 용이성** | UI 통합 테스트 중심 | 단위 테스트 용이 |

### 개발 흐름의 차이

**FSD 개발 흐름** (하향식):

```
1. 페이지 설계 → 2. 필요한 위젯 식별 → 3. features 구현 → 4. entities 정의 → 5. shared 추가
```

**Clean Architecture 개발 흐름** (상향식):

```
1. 도메인 엔티티 정의 → 2. Use Case 구현 → 3. Repository 인터페이스 → 4. API 어댑터 → 5. UI 연결
```

두 접근법은 상충하지 않는다. 오히려 보완적이다.

## 하이브리드 아키텍처 설계

### 핵심 아이디어

FSD의 레이어 구조는 유지하면서, `entities` 레이어에 Clean Architecture의 도메인/유스케이스/리포지토리 패턴을 적용한다.

```
src/
├── app/                    # FSD: 앱 초기화
├── pages/                  # FSD: 페이지 컴포넌트
├── widgets/                # FSD: 복합 UI 블록
├── features/               # FSD: 사용자 상호작용
├── entities/               # FSD + Clean Architecture 혼합
│   ├── order/
│   │   ├── domain/         # Clean: 순수 도메인 모델
│   │   ├── application/    # Clean: Use Case
│   │   ├── infrastructure/ # Clean: Repository 구현
│   │   └── ui/             # FSD: 엔티티 UI
│   └── user/
└── shared/                 # FSD: 공통 유틸리티
```

### 레이어별 상세 설계

#### 1. entities/{domain}/domain - 순수 도메인 계층

외부 의존성이 없는 순수 TypeScript 코드. 비즈니스 규칙과 검증 로직을 담는다.

```typescript
// entities/order/domain/order.entity.ts
export interface OrderItem {
  productId: string;
  productName: string;
  quantity: number;
  unitPrice: number;
}

export interface Order {
  id: string;
  customerId: string;
  items: OrderItem[];
  status: OrderStatus;
  createdAt: Date;
}

export type OrderStatus = 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';

// 도메인 로직: 총액 계산
export function calculateOrderTotal(order: Order): number {
  return order.items.reduce(
    (sum, item) => sum + item.quantity * item.unitPrice,
    0
  );
}

// 도메인 규칙: 주문 가능 여부 검증
export function canPlaceOrder(order: Order): { valid: boolean; reason?: string } {
  const total = calculateOrderTotal(order);

  if (order.items.length === 0) {
    return { valid: false, reason: '주문 항목이 비어있습니다' };
  }

  if (total < 10000) {
    return { valid: false, reason: '최소 주문 금액은 10,000원입니다' };
  }

  return { valid: true };
}
```

핵심 특징:
- React, Next.js, API 클라이언트 등 외부 라이브러리 import 없음
- 순수 함수로 구현하여 테스트 용이
- 비즈니스 규칙이 한 곳에 집중

#### 2. entities/{domain}/application - Use Case 계층

비즈니스 워크플로우를 조율한다. Repository 인터페이스에 의존하며, 구현체는 주입받는다.

```typescript
// entities/order/application/ports/order.repository.ts
import type { Order, OrderItem } from '../domain/order.entity';

// Repository 인터페이스 (포트)
export interface OrderRepository {
  findById(id: string): Promise<Order | null>;
  findByCustomer(customerId: string): Promise<Order[]>;
  save(order: Order): Promise<Order>;
  update(order: Order): Promise<Order>;
}

// 외부 서비스 인터페이스
export interface InventoryService {
  checkStock(productId: string, quantity: number): Promise<boolean>;
  reserveStock(productId: string, quantity: number): Promise<void>;
}

export interface PaymentService {
  processPayment(orderId: string, amount: number): Promise<{ success: boolean; transactionId?: string }>;
}
```

```typescript
// entities/order/application/use-cases/create-order.use-case.ts
import { Order, OrderItem, canPlaceOrder, calculateOrderTotal } from '../domain/order.entity';
import type { OrderRepository, InventoryService, PaymentService } from '../ports/order.repository';

interface CreateOrderInput {
  customerId: string;
  items: Omit<OrderItem, 'productName'>[];
}

interface CreateOrderResult {
  success: boolean;
  order?: Order;
  error?: string;
}

export class CreateOrderUseCase {
  constructor(
    private orderRepository: OrderRepository,
    private inventoryService: InventoryService,
    private paymentService: PaymentService
  ) {}

  async execute(input: CreateOrderInput): Promise<CreateOrderResult> {
    // 1. 도메인 객체 생성
    const order: Order = {
      id: crypto.randomUUID(),
      customerId: input.customerId,
      items: input.items.map(item => ({ ...item, productName: '' })), // 실제로는 조회 필요
      status: 'pending',
      createdAt: new Date(),
    };

    // 2. 도메인 규칙 검증
    const validation = canPlaceOrder(order);
    if (!validation.valid) {
      return { success: false, error: validation.reason };
    }

    // 3. 재고 확인
    for (const item of order.items) {
      const hasStock = await this.inventoryService.checkStock(item.productId, item.quantity);
      if (!hasStock) {
        return { success: false, error: `재고 부족: ${item.productId}` };
      }
    }

    // 4. 결제 처리
    const total = calculateOrderTotal(order);
    const payment = await this.paymentService.processPayment(order.id, total);
    if (!payment.success) {
      return { success: false, error: '결제 처리 실패' };
    }

    // 5. 재고 예약
    for (const item of order.items) {
      await this.inventoryService.reserveStock(item.productId, item.quantity);
    }

    // 6. 주문 저장
    const savedOrder = await this.orderRepository.save({
      ...order,
      status: 'confirmed',
    });

    return { success: true, order: savedOrder };
  }
}
```

핵심 특징:
- 복잡한 비즈니스 워크플로우를 단계별로 조율
- 인터페이스(Port)에만 의존, 구현체는 외부에서 주입
- 각 단계가 명확하여 디버깅 용이

#### 3. entities/{domain}/infrastructure - Repository 구현

실제 API 호출, 데이터 변환을 담당한다.

```typescript
// entities/order/infrastructure/order.api-repository.ts
import type { Order } from '../domain/order.entity';
import type { OrderRepository } from '../application/ports/order.repository';
import { apiClient } from '@/shared/api/client';

// API 응답 타입 (서버 스키마)
interface OrderApiResponse {
  order_id: string;
  customer_id: string;
  order_items: Array<{
    product_id: string;
    product_name: string;
    qty: number;
    price: number;
  }>;
  order_status: string;
  created_at: string;
}

// API 응답 → 도메인 모델 변환
function toDomain(response: OrderApiResponse): Order {
  return {
    id: response.order_id,
    customerId: response.customer_id,
    items: response.order_items.map(item => ({
      productId: item.product_id,
      productName: item.product_name,
      quantity: item.qty,
      unitPrice: item.price,
    })),
    status: response.order_status as Order['status'],
    createdAt: new Date(response.created_at),
  };
}

// 도메인 모델 → API 요청 변환
function toApi(order: Order): Partial<OrderApiResponse> {
  return {
    customer_id: order.customerId,
    order_items: order.items.map(item => ({
      product_id: item.productId,
      product_name: item.productName,
      qty: item.quantity,
      price: item.unitPrice,
    })),
    order_status: order.status,
  };
}

export class OrderApiRepository implements OrderRepository {
  async findById(id: string): Promise<Order | null> {
    try {
      const response = await apiClient.get<OrderApiResponse>(`/orders/${id}`);
      return toDomain(response.data);
    } catch (error) {
      if (error.status === 404) return null;
      throw error;
    }
  }

  async findByCustomer(customerId: string): Promise<Order[]> {
    const response = await apiClient.get<OrderApiResponse[]>('/orders', {
      params: { customer_id: customerId },
    });
    return response.data.map(toDomain);
  }

  async save(order: Order): Promise<Order> {
    const response = await apiClient.post<OrderApiResponse>('/orders', toApi(order));
    return toDomain(response.data);
  }

  async update(order: Order): Promise<Order> {
    const response = await apiClient.put<OrderApiResponse>(`/orders/${order.id}`, toApi(order));
    return toDomain(response.data);
  }
}
```

핵심 특징:
- API 응답 형태(snake_case)와 도메인 모델(camelCase) 분리
- 서버 API 변경 시 이 파일만 수정
- Repository 인터페이스를 구현하므로 테스트 시 Mock으로 교체 가능

#### 4. entities/{domain}/ui - 엔티티 UI (FSD 패턴 유지)

```typescript
// entities/order/ui/OrderCard.tsx
import type { Order } from '../domain/order.entity';
import { calculateOrderTotal } from '../domain/order.entity';
import { formatCurrency } from '@/shared/lib/format';

interface OrderCardProps {
  order: Order;
}

export function OrderCard({ order }: OrderCardProps) {
  const total = calculateOrderTotal(order);

  return (
    <div className="rounded-lg border p-4">
      <div className="flex justify-between">
        <span className="font-medium">주문 #{order.id.slice(0, 8)}</span>
        <OrderStatusBadge status={order.status} />
      </div>
      <p className="text-sm text-muted-foreground">
        {order.items.length}개 상품
      </p>
      <p className="text-lg font-bold">{formatCurrency(total)}</p>
    </div>
  );
}
```

#### 5. features - React Query와 Use Case 연결

features 레이어에서 Use Case를 React Query와 연결한다.

```typescript
// features/order/create-order/api/useCreateOrder.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { CreateOrderUseCase } from '@/entities/order/application/use-cases/create-order.use-case';
import { OrderApiRepository } from '@/entities/order/infrastructure/order.api-repository';
import { InventoryApiService } from '@/entities/inventory/infrastructure/inventory.api-service';
import { PaymentApiService } from '@/entities/payment/infrastructure/payment.api-service';

// 의존성 조립 (실제로는 DI 컨테이너 사용 권장)
const orderRepository = new OrderApiRepository();
const inventoryService = new InventoryApiService();
const paymentService = new PaymentApiService();
const createOrderUseCase = new CreateOrderUseCase(
  orderRepository,
  inventoryService,
  paymentService
);

export function useCreateOrder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: Parameters<typeof createOrderUseCase.execute>[0]) =>
      createOrderUseCase.execute(input),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['orders'] });
    },
  });
}
```

```typescript
// features/order/create-order/ui/CreateOrderButton.tsx
import { Button } from '@/shared/ui';
import { useCreateOrder } from '../api/useCreateOrder';
import { useCartStore } from '@/entities/cart';

export function CreateOrderButton() {
  const { items, customerId, clearCart } = useCartStore();
  const { mutate: createOrder, isPending, error } = useCreateOrder();

  const handleClick = () => {
    createOrder(
      { customerId, items },
      {
        onSuccess: (result) => {
          if (result.success) {
            clearCart();
            // 성공 처리
          } else {
            // 에러 처리
          }
        },
      }
    );
  };

  return (
    <Button onClick={handleClick} disabled={isPending}>
      {isPending ? '주문 처리 중...' : '주문하기'}
    </Button>
  );
}
```

### 의존성 주입 설정

대규모 프로젝트에서는 DI 컨테이너를 사용하여 의존성을 관리한다.

```typescript
// shared/di/container.ts
import { createContainer, asClass, asValue } from 'awilix';

// Repository 구현체
import { OrderApiRepository } from '@/entities/order/infrastructure/order.api-repository';
import { InventoryApiService } from '@/entities/inventory/infrastructure/inventory.api-service';
import { PaymentApiService } from '@/entities/payment/infrastructure/payment.api-service';

// Use Cases
import { CreateOrderUseCase } from '@/entities/order/application/use-cases/create-order.use-case';
import { GetOrdersUseCase } from '@/entities/order/application/use-cases/get-orders.use-case';

const container = createContainer();

container.register({
  // Repositories
  orderRepository: asClass(OrderApiRepository).singleton(),
  inventoryService: asClass(InventoryApiService).singleton(),
  paymentService: asClass(PaymentApiService).singleton(),

  // Use Cases
  createOrderUseCase: asClass(CreateOrderUseCase),
  getOrdersUseCase: asClass(GetOrdersUseCase),
});

export { container };
```

```typescript
// features/order/create-order/api/useCreateOrder.ts
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { container } from '@/shared/di/container';

export function useCreateOrder() {
  const queryClient = useQueryClient();
  const createOrderUseCase = container.resolve('createOrderUseCase');

  return useMutation({
    mutationFn: createOrderUseCase.execute.bind(createOrderUseCase),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['orders'] });
    },
  });
}
```

## Next.js App Router 통합

Next.js App Router와 하이브리드 아키텍처를 통합하는 폴더 구조:

```
project-root/
├── app/                          # Next.js 라우팅
│   ├── orders/
│   │   ├── page.tsx             # FSD 페이지 import
│   │   └── [id]/
│   │       └── page.tsx
│   └── layout.tsx
├── pages/                        # 빈 폴더 (빌드 오류 방지)
│   └── .gitkeep
└── src/
    ├── app/                      # FSD app 레이어
    │   ├── providers/
    │   │   ├── query.tsx
    │   │   ├── di.tsx           # DI Provider
    │   │   └── index.tsx
    │   └── styles/
    ├── pages/                    # FSD pages 레이어
    │   └── order/
    │       └── ui/
    │           ├── OrderListPage.tsx
    │           └── OrderDetailPage.tsx
    ├── widgets/
    ├── features/
    │   └── order/
    │       ├── create-order/
    │       └── cancel-order/
    ├── entities/
    │   ├── order/
    │   │   ├── domain/          # 순수 도메인
    │   │   ├── application/     # Use Cases
    │   │   ├── infrastructure/  # Repository 구현
    │   │   ├── ui/              # 엔티티 UI
    │   │   └── index.ts
    │   ├── user/
    │   └── product/
    └── shared/
        ├── api/
        ├── di/                   # DI 컨테이너
        ├── ui/
        └── lib/
```

```typescript
// app/orders/page.tsx
import { OrderListPage } from '@/pages/order';

export default function OrdersRoute() {
  return <OrderListPage />;
}

export const metadata = {
  title: '주문 목록',
};
```

## 테스트 전략

하이브리드 아키텍처의 가장 큰 장점은 테스트 용이성이다.

### 도메인 로직 단위 테스트

```typescript
// entities/order/domain/__tests__/order.entity.test.ts
import { calculateOrderTotal, canPlaceOrder, Order } from '../order.entity';

describe('Order Domain', () => {
  const createOrder = (items: Order['items']): Order => ({
    id: 'test-id',
    customerId: 'customer-1',
    items,
    status: 'pending',
    createdAt: new Date(),
  });

  describe('calculateOrderTotal', () => {
    it('should calculate total correctly', () => {
      const order = createOrder([
        { productId: '1', productName: 'A', quantity: 2, unitPrice: 5000 },
        { productId: '2', productName: 'B', quantity: 1, unitPrice: 3000 },
      ]);

      expect(calculateOrderTotal(order)).toBe(13000);
    });
  });

  describe('canPlaceOrder', () => {
    it('should reject empty order', () => {
      const order = createOrder([]);
      const result = canPlaceOrder(order);

      expect(result.valid).toBe(false);
      expect(result.reason).toContain('비어있습니다');
    });

    it('should reject order under minimum amount', () => {
      const order = createOrder([
        { productId: '1', productName: 'A', quantity: 1, unitPrice: 5000 },
      ]);
      const result = canPlaceOrder(order);

      expect(result.valid).toBe(false);
      expect(result.reason).toContain('10,000원');
    });
  });
});
```

### Use Case 테스트 (Mock Repository)

```typescript
// entities/order/application/__tests__/create-order.use-case.test.ts
import { CreateOrderUseCase } from '../use-cases/create-order.use-case';

describe('CreateOrderUseCase', () => {
  const mockOrderRepository = {
    save: jest.fn(),
    findById: jest.fn(),
    findByCustomer: jest.fn(),
    update: jest.fn(),
  };

  const mockInventoryService = {
    checkStock: jest.fn(),
    reserveStock: jest.fn(),
  };

  const mockPaymentService = {
    processPayment: jest.fn(),
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should create order successfully', async () => {
    mockInventoryService.checkStock.mockResolvedValue(true);
    mockPaymentService.processPayment.mockResolvedValue({ success: true, transactionId: 'tx-1' });
    mockOrderRepository.save.mockImplementation(order => Promise.resolve(order));

    const useCase = new CreateOrderUseCase(
      mockOrderRepository,
      mockInventoryService,
      mockPaymentService
    );

    const result = await useCase.execute({
      customerId: 'customer-1',
      items: [{ productId: 'p1', quantity: 2, unitPrice: 10000 }],
    });

    expect(result.success).toBe(true);
    expect(result.order?.status).toBe('confirmed');
    expect(mockInventoryService.reserveStock).toHaveBeenCalled();
  });

  it('should fail when stock is insufficient', async () => {
    mockInventoryService.checkStock.mockResolvedValue(false);

    const useCase = new CreateOrderUseCase(
      mockOrderRepository,
      mockInventoryService,
      mockPaymentService
    );

    const result = await useCase.execute({
      customerId: 'customer-1',
      items: [{ productId: 'p1', quantity: 100, unitPrice: 10000 }],
    });

    expect(result.success).toBe(false);
    expect(result.error).toContain('재고 부족');
    expect(mockPaymentService.processPayment).not.toHaveBeenCalled();
  });
});
```

## 언제 하이브리드 아키텍처를 적용해야 하는가

### 적합한 경우

- **복잡한 비즈니스 로직**: 주문/결제/정산 등 여러 단계를 거치는 워크플로우
- **도메인 규칙이 자주 변경**: 비즈니스 정책 변경에 빠르게 대응해야 하는 경우
- **테스트 커버리지 중요**: 핵심 비즈니스 로직에 대한 단위 테스트가 필수인 경우
- **API 스키마 불안정**: 백엔드 API가 자주 변경되는 초기 개발 단계
- **장기 유지보수**: 2년 이상 운영될 제품

### 부적합한 경우

- **단순 CRUD 애플리케이션**: 데이터를 조회/수정만 하는 관리자 도구
- **소규모 프로젝트**: 1-2명이 단기간 개발하는 프로젝트
- **프로토타입/MVP**: 빠른 검증이 목적인 초기 단계
- **정적 콘텐츠 중심**: 랜딩 페이지, 블로그 등

### 점진적 도입 전략

모든 엔티티에 Clean Architecture를 적용할 필요는 없다. 복잡도가 높은 핵심 도메인부터 시작한다.

```
entities/
├── order/          # 하이브리드 적용 (복잡한 비즈니스 로직)
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── ui/
├── product/        # FSD 기본 구조 (단순 CRUD)
│   ├── model/
│   ├── api/
│   └── ui/
└── user/           # FSD 기본 구조
    ├── model/
    ├── api/
    └── ui/
```

## 참고 자료

### FSD + Clean Architecture

- [Feature-Sliced Design vs Clean Architecture](https://philrich.dev/fsd-vs-clean-architecture/) - 두 아키텍처 비교
- [Clean Architecture vs. Feature-Sliced Design in Next.js Applications](https://medium.com/@metastability/clean-architecture-vs-feature-sliced-design-in-next-js-applications-04df25e62690) - Next.js 적용 사례

### Clean Architecture

- [Clean Architecture on Frontend](https://bespoyasov.me/blog/clean-architecture-on-frontend/) - 프론트엔드 클린 아키텍처 상세 가이드
- [GitHub: frontend-clean-architecture](https://github.com/bespoyasov/frontend-clean-architecture) - React + TypeScript 예제
- [GitHub: react-with-clean-architecture](https://github.com/falsy/react-with-clean-architecture) - 레이어별 구현 예제

### 관련 블로그

- [Feature-Sliced Design: 프론트엔드 아키텍처의 표준화된 접근법](https://blog.imprun.dev/70) - FSD 기본 개념
- [AI Agent를 위한 Frontend 개발 가이드](https://blog.imprun.dev/68) - AGENTS.md와 FSD 조합
