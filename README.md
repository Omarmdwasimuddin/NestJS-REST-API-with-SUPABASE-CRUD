# NestJS REST API with Supabase CRUD 

আগের গাইডে Supabase PostgreSQL-কে NestJS-এর সাথে connect করা হয়েছিল। এবার সেই connection ব্যবহার করে একটা পুরোপুরি কাজ করা **CRUD REST API** বানানো হবে — `Employee` নামের একটা entity দিয়ে, যেখানে সাধারণ CRUD ছাড়াও **search আর filter**-ও থাকবে।

---

## ধাপ ১: Project তৈরি করা

PowerShell-এ দাও:

```bash
nest new nest-supabase
```

```bash
cd nest-supabase
```

```bash
code .
```

(শেষ command দিয়ে VS Code-এ project খুলে যাবে)

---

## ধাপ ২: Supabase PostgreSQL Connect করা

এই ধাপটা আগেই আলাদা গাইডে বিস্তারিত দেখানো হয়েছে — Supabase project তৈরি, `.env`-এ `DATABASE_URL` বসানো, `@nestjs/typeorm`/`typeorm`/`pg` install করা ইত্যাদি। বিস্তারিত দেখো: [Connect Supabase PostgreSQL with NestJS](https://github.com/Omarmdwasimuddin/Connect-Supabase-PostgreSQL-with-NestJS)।

---

## ধাপ ৩: Module, Service, Controller তৈরি করা

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

## ধাপ ৪: Entity File তৈরি করা

`employees.entity.ts` নামের file manually বানাতে হবে।

### `employees.entity.ts`

```ts
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

চারটা column: `name`, `position`, `department`, `salary` — সাথে auto-generated `id`।

---

## ধাপ ৫: Module-এ Entity Register করা

### `employee.module.ts`

```ts
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

`TypeOrmModule.forFeature([Employee])` দিয়ে `Employee` entity-র repository এই module-এর ভিতরে inject করার জন্য available হয়ে যায়।

---

## ধাপ ৬: `app.module.ts` Setup করা

> **নোট:** `app.module.ts`-এ যোগ করতে হবে:
> - `ConfigModule.forRoot()`
> - `TypeOrmModule.forRoot({ type: 'postgres', url: process.env.DATABASE_URL, autoLoadEntities: true, synchronize: true })`

### `app.module.ts`

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

---

## ধাপ ৭: Service লেখা (Business Logic)

### `employee.service.ts`

```ts
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Employee } from './employees.entity';
import { ILike, Like, Repository } from 'typeorm';

@Injectable()
export class EmployeeService {
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
        const updatedEmployee = Object.assign(employee, updateData);
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
            query.andWhere('employee.name ILIKE :name', { name: `%${filters.name}%` });
        }
        if (filters.department) {
            query.andWhere(`employee.department ILIKE :department`, { department: `%${filters.department}%` });
        }
        return query.getMany();
    }
}
```

### প্রতিটা Method কী করছে

| Method | কাজ |
|---|---|
| `create()` | নতুন employee তৈরি ও save করে |
| `findAll()` | সবগুলো employee fetch করে |
| `findOne()` | নির্দিষ্ট `id` দিয়ে employee খোঁজে, না পেলে `NotFoundException` |
| `findByName()` | Exact নাম match করে খোঁজে (case-sensitive, পুরো নাম মিলতে হবে) |
| `searchByName()` | আংশিক নাম match করে খোঁজে (`ILIKE` দিয়ে case-insensitive) |
| `update()` | Employee খুঁজে বের করে, `Object.assign()` দিয়ে পুরোনো data-র উপর নতুন data বসিয়ে save করে |
| `delete()` | নির্দিষ্ট `id`-এর employee মুছে ফেলে, কিছু না মুছলে (`affected === 0`) `NotFoundException` |
| `search()` | `createQueryBuilder()` দিয়ে dynamic filter — `name` আর `department` দুটোই optional, যেটা দেওয়া হবে সেটাই filter-এ যোগ হবে |

### `Like` vs `ILike` — পার্থক্য

- **`Like`** — case-sensitive (বড়/ছোট হাতের অক্ষর মিলতে হবে)
- **`ILike`** — case-insensitive (বড়/ছোট হাতের অক্ষর নিয়ে চিন্তা নেই)

এই কারণেই `searchByName()`-এ `Like`-এর বদলে `ILike` ব্যবহার করা হয়েছে — যাতে user "John", "john", "JOHN" যেভাবেই লিখুক, খুঁজে পায়।

### `search()` method-এ Dynamic Query Builder

`createQueryBuilder()` ব্যবহার করার সুবিধা হলো — কোন filter দেওয়া হয়েছে সেটার উপর ভিত্তি করে ধাপে ধাপে (`andWhere()`) query তৈরি করা যায়। শুধু `name` দিলে শুধু name দিয়ে filter হবে, দুটোই দিলে দুটো condition-ই AND দিয়ে যুক্ত হবে, কিছুই না দিলে সবগুলো employee ফেরত আসবে।

---

## ধাপ ৮: Controller লেখা (Route Handler)

> **গুরুত্বপূর্ণ নোট:** `GET` method-এর ভিতরে **`:id`-এর আগেই** filter/search-এর মতো নির্দিষ্ট নামের route (যেমন `filter`, `search/:keyword`) বসাতে হবে — না হলে filter route কাজ করবে না। কারণ NestJS route-গুলো উপর থেকে নিচে ক্রম অনুযায়ী match করে; `:id` একটা wildcard-এর মতো, তাই এটা যদি আগে থাকে, `/employee-bd/filter`-এর "filter" শব্দটাকেও `id` হিসেবে ধরে ফেলবে।

### `employee.controller.ts`

```ts
import { Body, Controller, Delete, Get, Param, Post, Put, Query } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { Employee } from './employees.entity';

@Controller('employee-bd')
export class EmployeeController {
    constructor(private readonly employeeService: EmployeeService) {}

    @Post()
    async createEmployee(@Body() employeeData: Partial<Employee>) {
        return this.employeeService.create(employeeData);
    }

    @Get()
    async findAllEmployees(): Promise<Employee[]> {
        return this.employeeService.findAll();
    }

    @Get('filter')
    async filterEmployees(
        @Query('name') name?: string,
        @Query('department') department?: string,
    ): Promise<Employee[]> {
        return this.employeeService.search({ name, department });
    }

    // find by id
    @Get(':id')
    async findEmployeeById(@Param('id') id: number): Promise<Employee> {
        return this.employeeService.findOne(id);
    }

    // find by name
    @Get('name/:name')
    async findEmployeeByName(@Param('name') name: string): Promise<Employee[]> {
        return this.employeeService.findByName(name);
    }

    // search by name (partial match)
    @Get('search/:keyword')
    async searchEmployeeByName(@Param('keyword') keyword: string): Promise<Employee[]> {
        return this.employeeService.searchByName(keyword);
    }

    @Put(':id')
    async updateEmployee(
        @Param('id') id: number,
        @Body() updateData: Partial<Employee>): Promise<Employee> {
        return this.employeeService.update(id, updateData);
    }

    @Delete(':id')
    async deleteEmployee(@Param('id') id: number): Promise<{ message: string }> {
        return this.employeeService.delete(id);
    }
}
```

> ⚠️ **একটা bug ঠিক করে দেওয়া হয়েছে:** Original code-এ `employee.module.ts`-এ service-এর নাম ছিল `EmployeeService` (import path `./employee.service`), কিন্তু `employee.controller.ts`-এ import করা হচ্ছিল `EmployeeBdService` নামে, `./employee-bd.service` নামের একটা আলাদা file থেকে — যেটা আসলে কোথাও define করাই ছিল না। এই mismatch থাকলে TypeScript compile error দেবে ("Cannot find module") অথবা DI resolve করতে ব্যর্থ হবে। উপরের code-এ পুরোটাই একই নাম (`EmployeeService`) ও একই path (`./employee.service`) দিয়ে সামঞ্জস্যপূর্ণ করে দেওয়া হয়েছে। **নিজের project-এ কাজ করার সময় লক্ষ্য রেখো — module, service, controller-এ সবখানে ঠিক একই class name আর import path ব্যবহার হচ্ছে কিনা।**

### Route-গুলো একনজরে

| HTTP Method | Route | কাজ |
|---|---|---|
| `POST` | `/employee-bd` | নতুন employee তৈরি |
| `GET` | `/employee-bd` | সবগুলো employee দেখা |
| `GET` | `/employee-bd/filter?name=...&department=...` | Dynamic filter (name/department, দুটোই optional) |
| `GET` | `/employee-bd/:id` | নির্দিষ্ট `id`-এর employee দেখা |
| `GET` | `/employee-bd/name/:name` | Exact নাম দিয়ে খোঁজা |
| `GET` | `/employee-bd/search/:keyword` | আংশিক নাম দিয়ে খোঁজা (case-insensitive) |
| `PUT` | `/employee-bd/:id` | Employee-এর data update করা |
| `DELETE` | `/employee-bd/:id` | Employee মুছে ফেলা |

---
