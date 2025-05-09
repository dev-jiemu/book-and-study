### NestJS : Jest

```npm
npm i --save-dev @nestjs/testing
```

1. Unit test
example)
```typescript
import { Test } from '@nestjs/testing';
import { CatsController } from './cats.controller';
import { CatsService } from './cats.service';

describe('CatsController', () => {
    let catsController: CatsController;
    let catsService: CatsService;

    // 테스팅용 모듈 세팅
    beforeEach(async () => {
        // 인스턴스 처리
        const moduleRef = await Test.createTestingModule({
            controllers: [CatsController],
            providers: [CatsService],
        }).compile();

        catsService = moduleRef.get(CatsService);
        catsController = moduleRef.get(CatsController);
    });

    describe('findAll', () => {
        it('should return an array of cats', async () => {
            const result = ['test'];
            jest.spyOn(catsService, 'findAll').mockImplementation(() => result);

            expect(await catsController.findAll()).toBe(result);
        });
    });
});
```

2. 모듈 : 종속성 오버라이드
테스트 간소화를 하기 위해 모의 구현도 가능
```typescript
const moduleRef = await Test.createTestingModule({
  imports: [CatsModule],
})
.overrideProvider(CatsService)
.useValue(mockCatsService) // 이걸로 대체
.compile();
```

3. E2E 테스팅 하고 싶으면
```typescript
// cats.e2e-spec.ts
import * as request from 'supertest';
import { Test } from '@nestjs/testing';
import { CatsModule } from '../../src/cats/cats.module';
import { CatsService } from '../../src/cats/cats.service';
import { INestApplication } from '@nestjs/common';

describe('Cats', () => {
  let app: INestApplication;
  let catsService = { findAll: () => ['test'] };

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [CatsModule],
    })
    .overrideProvider(CatsService)
    .useValue(catsService)
    .compile();

    app = moduleRef.createNestApplication();
    await app.init();
  });

  it(`/GET cats`, () => {
    return request(app.getHttpServer())
      .get('/cats')
      .expect(200)
      .expect({ data: catsService.findAll() });
  });

  afterAll(async () => {
    await app.close();
  });
});
```

Ref.
- https://github.com/jestjs/jest 
- https://docs.nestjs.com/fundamentals/testing