# MongoDB
### MongoDB Study with Nest.js

#### unique key
MongoDB 에서는 `각 필드에 대해 개별적으로 unique 제약 조건이 적용`됨
```typescript
@Schema()
export class Station {
  @Prop({ required: true, unique: true })
  stationManageNo: string // 정류소코드

  @Prop()
  cityCode: string

  @Prop()
  cityName: string

  @Prop({ required: true, unique: true })
  stationName: string
}
```
=> 스키마를 이렇게 설정하면 RDS 에서 둘다 겹쳤을때 에러가 발생하는 줄 알았더니.. 별개라고 함...

조금 복잡한 스키마를 설정하고 싶다면 이렇게
```typescript
export const StationSchema = SchemaFactory.createForClass(Station)
StationSchema.index({ stationManageNo: 1, stationName: 1 }, { unique: true })
export type StationDocument = Station & Document
```

---
#### Update, Bulk 차이
```typescript
// 조건에 일치하는 데이터 한번에 업데이트 처리함
db.collection.updateMany(
  { status: "pending" },
  { $set: { status: "approved" } }
);

// 다양한 작업 한번에 묶어서 하고싶으면 bulk
db.collection.bulkWrite([
    { updateMany: {
            filter: { status: "pending" },
            update: { $set: { status: "approved" } }
        }
    },
    { updateOne: {
            filter: { _id: ObjectId("...") },
            update: { $inc: { count: 1 } }
        }
    },
    { deleteOne: {
            filter: { expired: true }
        }
    }
]);
```
| 항목             | `updateMany()`                         | `bulkWrite()`                             |
|------------------|-----------------------------------------|--------------------------------------------|
| **사용 목적**    | 조건에 맞는 여러 문서 업데이트         | 다양한 작업을 한 번에 처리                |
| **작업 종류**    | `update`만 가능                         | `insert`, `updateOne`, `updateMany`, `delete` 등 가능 |
| **작업 수**      | 한 번에 하나의 조건                    | 여러 조건과 여러 작업 가능                |
| **유연성**       | 낮음                                    | 높음                                       |
| **성능**         | 단일 `update` 작업에 적합               | 대량 작업 시 성능 최적화 가능             |

