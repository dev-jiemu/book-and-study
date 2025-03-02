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