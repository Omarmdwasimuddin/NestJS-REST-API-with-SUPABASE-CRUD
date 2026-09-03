## NestJS REST API with SUPABASE CRUD

#### Powershell e daw
```bash
nest new nest-supabase
```
```bash
cd nest-supabase
```
```bash
code .
```
---


#### [Connect Supabase PostgreSQL with NestJS](https://github.com/Omarmdwasimuddin/Connect-Supabase-PostgreSQL-with-NestJS)

#### Create module, service, controller
```bash
nest g module employee
```
```bash
nest g service employee
```
```bash
nest g controller employee
```
---


>#### file add koro- employees.entity.ts
#### `employees.entity.ts`
```bash
import { Entity, PrimaryGeneratedColumn, Column } from "typeorm";


@Entity()
export class Employee {
    @PrimaryGeneratedColumn()
    id!: number;

    @Column()
    name!: string;

    @Column()
    position!: string;

    @Column()
    department!: string;

    @Column()
    salary!: number;
}
```
---


>#### add- imports: [TypeOrmModule.forFeature([Employee])],

#### `employee-bd.module.ts`
```bash
import { Module } from '@nestjs/common';
import { EmployeeBdService } from './employee-bd.service';
import { EmployeeBdController } from './employee-bd.controller';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Employee } from './employees.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Employee])],
  providers: [EmployeeBdService],
  controllers: [EmployeeBdController]
})
export class EmployeeBdModule {}
```
