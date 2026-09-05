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
import { ILike, Repository } from 'typeorm';

@Injectable()
export class EmployeeService {
    constructor(
        @InjectRepository(Employee) private employeeRepository: Repository<Employee>,
    ) {}

    async createEmployee(employeeData: Partial<Employee>): Promise<Employee> {
        const employee = this.employeeRepository.create(employeeData);
        return this.employeeRepository.save(employee);
    }

    async findAllEmployees(): Promise<Employee[]> {
        return this.employeeRepository.find();
    }

    async findOneEmployee(id: number): Promise<Employee> {
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
    async searchByKeyword(keyword: string): Promise<Employee[]> {
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
| `createEmployee()` | নতুন employee তৈরি ও save করে |
| `findAllEmployees()` | সবগুলো employee fetch করে |
| `findOneEmployee()` | নির্দিষ্ট `id` দিয়ে employee খোঁজে, না পেলে `NotFoundException` |
| `findByName()` | Exact নাম match করে খোঁজে (case-sensitive, পুরো নাম মিলতে হবে) |
| `searchByKeyword()` | আংশিক নাম match করে খোঁজে (`ILike` দিয়ে case-insensitive) |
| `update()` | Employee খুঁজে বের করে, `Object.assign()` দিয়ে পুরোনো data-র উপর নতুন data বসিয়ে save করে |
| `delete()` | নির্দিষ্ট `id`-এর employee মুছে ফেলে, কিছু না মুছলে (`affected === 0`) `NotFoundException` |
| `search()` | `createQueryBuilder()` দিয়ে dynamic filter — `name` আর `department` দুটোই optional, যেটা দেওয়া হবে সেটাই filter-এ যোগ হবে |

### `Like` vs `ILike` — পার্থক্য

- **`Like`** — case-sensitive (বড়/ছোট হাতের অক্ষর মিলতে হবে)
- **`ILike`** — case-insensitive (বড়/ছোট হাতের অক্ষর নিয়ে চিন্তা নেই)

এই কারণেই `searchByKeyword()`-এ `Like`-এর বদলে `ILike` ব্যবহার করা হয়েছে — যাতে user "John", "john", "JOHN" যেভাবেই লিখুক, খুঁজে পায়।

### `search()` method-এ Dynamic Query Builder

`createQueryBuilder()` ব্যবহার করার সুবিধা হলো — কোন filter দেওয়া হয়েছে সেটার উপর ভিত্তি করে ধাপে ধাপে (`andWhere()`) query তৈরি করা যায়। শুধু `name` দিলে শুধু name দিয়ে filter হবে, দুটোই দিলে দুটো condition-ই AND দিয়ে যুক্ত হবে, কিছুই না দিলে সবগুলো employee ফেরত আসবে।

---

## ধাপ ৮: Controller লেখা (Route Handler)

> **গুরুত্বপূর্ণ নোট:** `GET` method-এর ভিতরে **`:id`-এর আগেই** filter-এর মতো নির্দিষ্ট নামের route বসাতে হবে — না হলে filter route কাজ করবে না। কারণ NestJS route-গুলো উপর থেকে নিচে ক্রম অনুযায়ী match করে; `:id` একটা wildcard-এর মতো, তাই এটা যদি আগে থাকে, `/employee/filter`-এর "filter" শব্দটাকেও `id` হিসেবে ধরে ফেলবে। নিচের code-এ ঠিক এই কারণেই `filter` route-টা `:id` route-এর **আগে** বসানো হয়েছে।

### `employee.controller.ts`

```ts
import { Body, Controller, Delete, Get, Param, Post, Put, Query } from '@nestjs/common';
import { EmployeeService } from './employee.service';
import { Employee } from './employees.entity';

@Controller('employee')
export class EmployeeController {
    constructor(private readonly employeeService: EmployeeService) {}

    @Post()
    async createEmployee(@Body() employeeData: Partial<Employee>) {
        return this.employeeService.createEmployee(employeeData);
    }

    @Get()
    async findAllEmployees(): Promise<Employee[]> {
        return this.employeeService.findAllEmployees();
    }

    @Get('filter')
    async filterEmployees(
        @Query('name') name?: string,
        @Query('department') department?: string,
    ): Promise<Employee[]> {
        return this.employeeService.search({ name, department });
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
    @Get('search/:keyword')
    async searchEmployeeByKeyword(@Param('keyword') keyword: string): Promise<Employee[]> {
        return this.employeeService.searchByKeyword(keyword);
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

### Route-গুলো একনজরে (ক্রম অনুযায়ী, যেভাবে register করতে হবে)

| ক্রম | HTTP Method | Route | কাজ |
|---|---|---|---|
| ১ | `POST` | `/employee` | নতুন employee তৈরি |
| ২ | `GET` | `/employee` | সবগুলো employee দেখা |
| ৩ | `GET` | `/employee/filter?name=...&department=...` | Dynamic filter (name/department, দুটোই optional) — **`:id`-এর আগে থাকতে হবে** |
| ৪ | `GET` | `/employee/:id` | নির্দিষ্ট `id`-এর employee দেখা |
| ৫ | `GET` | `/employee/name/:name` | Exact নাম দিয়ে খোঁজা |
| ৬ | `GET` | `/employee/search/:keyword` | আংশিক নাম দিয়ে খোঁজা (case-insensitive) |
| — | `PUT` | `/employee/:id` | Employee-এর data update করা |
| — | `DELETE` | `/employee/:id` | Employee মুছে ফেলা |

> **কেন `name/:name` আর `search/:keyword` কাজ করে, কিন্তু `filter` route-টা `:id`-এর পরে থাকলে কাজ করত না?**
> কারণ `name/:name` আর `search/:keyword`-এ route-এর শুরুতে একটা fixed segment (`name`, `search`) আছে, যেটা `:id`-এর single-segment pattern-এর সাথে মেলে না (এগুলোতে দুটো segment থাকে)। কিন্তু `filter`-এ একটাই segment (`/employee/filter`) — তাই এটা যদি `:id`-এর পরে থাকত, NestJS প্রথমে `:id` route-টাই match করে ফেলত, `filter`-কে একটা `id`-এর value ধরে নিয়ে।

---
