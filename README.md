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

#### `employee.module.ts`
```bash
import { Module } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { EmployeeController } from './employee.controller';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Employee } from './employees.entity';

@Module({
  imports: [TypeOrmModule.forFeature([Employee])],
  providers: [EmployeeService],
  controllers: [EmployeeController]
})
export class EmployeeModule {}
```


> **Note:** `app.module.ts`- e add koro:
> - `ConfigModule.forRoot()`
> - `TypeOrmModule.forRoot({ type: 'postgres', url: process.env.DATABASE_URL, autoLoadEntities: true, synchronize: true })`

#### `app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { AppService } from './app.service';
import { EmployeeModule } from './employee/employee.module';
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';


@Module({
  imports: [ ConfigModule.forRoot(), TypeOrmModule.forRoot({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    autoLoadEntities: true,
    synchronize: true,
  }), EmployeeModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

#

#### `employee.service.ts`
```bash
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Employee } from './employees.entity';
import { ILike, Like, Repository } from 'typeorm';

@Injectable()
export class EmployeeBdService {
    constructor(
        @InjectRepository(Employee) private employeeRepository: Repository<Employee>,
    ) {}

    async create(employeeData: Partial<Employee>): Promise<Employee> {
        const employee = this.employeeRepository.create(employeeData);
        return this.employeeRepository.save(employee);
    }

    async findAll(): Promise<Employee[]> {
        return this.employeeRepository.find();
    }

    async findOne(id: number): Promise<Employee> {
        const employee = await this.employeeRepository.findOneBy({ id });
        if (!employee) {
            throw new NotFoundException(`Employee with ID ${id} not found`);
        }
        return employee;
    }

    async findByName(name: string): Promise<Employee[]> {
        const employee = await this.employeeRepository.find({
            where: { name },
        });
        if (!employee.length) {
            throw new NotFoundException(`Employee with name ${name} not found`);
        }
        return employee;
    }

    // Like is case-sensitive, ILike is case-insensitive
    async searchByName(keyword: string): Promise<Employee[]> {
        return this.employeeRepository.find({
            where: {
                //name: Like(`%${keyword}%`)
                name: ILike(`%${keyword}%`)
            }
        });
    }

    async update(id: number, updateData: Partial<Employee>): Promise<Employee> {
        const employee = await this.employeeRepository.findOneBy({ id });
        if (!employee) {
            throw new NotFoundException(`Employee with ID ${id} not found`);
        }
        const updatedEmployee = Object.assign(employee,  updateData);
        return this.employeeRepository.save(updatedEmployee);
    }

    async delete(id: number): Promise<{ message: string }> {
        const result = await this.employeeRepository.delete({ id });
        if (result.affected === 0) {
            throw new NotFoundException(`Employee with ID ${id} not found`);
        }
        return { message: `Employee with ID ${id} deleted successfully` };
    }

    async search(filters: { name?: string; department?: string }): Promise<Employee[]> {
        const query = this.employeeRepository.createQueryBuilder('employee');

        if (filters.name) {
            query.andWhere('employee.name ILIKE :name', { name: `%${filters.name}%`});
        }
        if (filters.department){
            query.andWhere(`employee.department ILIKE :department`, { department: `%${filters.department}%`});
        }
        return query.getMany();
    }

}
```

#

#### `employee.controller.ts`
```bash
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { Employee } from './employees.entity';

@Controller('employee')
export class EmployeeController {
    constructor(private readonly employeeService: EmployeeService){}

    @Post()
    async createEmployee(@Body() employeeData: Partial<Employee>){
        return this.employeeService.createEmployee(employeeData);
    }

    @Get()
    async findAllEmployees(): Promise<Employee[]> {
        return this.employeeService.findAllEmployees();
    }

    @Get(':id')
    async findEmployeeById(@Param('id') id: number): Promise<Employee | null> {
        return this.employeeService.findOneEmployee(id);
    }

    // find by name
    @Get('name/:name')
    async findEmployeeByName(@Param('name') name: string): Promise<Employee[]> {
        return this.employeeService.findByName(name);
    }

    // search by keyword (partial match)
    @Get(`search/:keyword`)
    async searchEmployeeByKeyword(@Param('keyword') keyword: string): Promise<Employee[]> {
        return this.employeeService.searchByKeyword(keyword);
    }
}
```
---
