# 📖 Chapter 4: Authentication (JWT & Bcrypt)

> **Tujuan:** Mengimplementasikan sistem autentikasi menggunakan JWT (JSON Web Token), melindungi endpoint API dengan Auth Guard, mengamankan password dengan bcrypt hashing, dan mengintegrasikan auth di Swagger.

---

## 📋 Daftar Isi

1. [Pendahuluan](#pendahuluan)
2. [Bagian A: Setup Auth Module](#bagian-a-setup-auth-module)
   - [Langkah 1: Generate Auth Resource](#langkah-1-generate-auth-resource)
   - [Langkah 2: Install & Konfigurasi Passport + JWT](#langkah-2-install--konfigurasi-passport--jwt)
3. [Bagian B: Membuat Login Endpoint](#bagian-b-membuat-login-endpoint)
   - [Langkah 3: Membuat Login DTO & Auth Entity](#langkah-3-membuat-login-dto--auth-entity)
   - [Langkah 4: Implementasi Login di AuthService](#langkah-4-implementasi-login-di-authservice)
   - [Langkah 5: Membuat Login Controller](#langkah-5-membuat-login-controller)
4. [Bagian C: Melindungi Endpoint dengan JWT](#bagian-c-melindungi-endpoint-dengan-jwt)
   - [Langkah 6: Membuat JWT Strategy](#langkah-6-membuat-jwt-strategy)
   - [Langkah 7: Membuat JWT Auth Guard](#langkah-7-membuat-jwt-auth-guard)
   - [Langkah 8: Menerapkan Guard ke Endpoint](#langkah-8-menerapkan-guard-ke-endpoint)
5. [Bagian D: Integrasi Auth di Swagger](#bagian-d-integrasi-auth-di-swagger)
   - [Langkah 9: Konfigurasi Bearer Auth di Swagger](#langkah-9-konfigurasi-bearer-auth-di-swagger)
6. [Bagian E: Hashing Password dengan Bcrypt](#bagian-e-hashing-password-dengan-bcrypt)
   - [Langkah 10: Hash Password saat Create & Update User](#langkah-10-hash-password-saat-create--update-user)
   - [Langkah 11: Update Seed Script](#langkah-11-update-seed-script)
   - [Langkah 12: Update Login untuk Bcrypt](#langkah-12-update-login-untuk-bcrypt)
7. [Ringkasan](#ringkasan)

---

## Pendahuluan

### Apa itu Authentication?

**Authentication** = Proses memverifikasi **siapa** kamu (identitas).

```
┌─────────────────────────────────────────────────────┐
│                  ALUR AUTHENTICATION                 │
│                                                      │
│  1. Client kirim email + password                    │
│           │                                          │
│           ▼                                          │
│  2. Server verifikasi credentials                    │
│           │                                          │
│           ▼                                          │
│  3. Server berikan JWT Token                         │
│           │                                          │
│           ▼                                          │
│  4. Client simpan token & kirim di setiap request    │
│           │                                          │
│           ▼                                          │
│  5. Server validasi token → izinkan/tolak akses      │
└─────────────────────────────────────────────────────┘
```

### Apa itu JWT (JSON Web Token)?

JWT adalah string token yang berisi informasi terenkripsi:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.     ← Header (algoritma)
eyJ1c2VySWQiOjF9.                            ← Payload (data: userId)
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c ← Signature (tanda tangan)
```

| Komponen | Isi |
|----------|-----|
| **Header** | Algoritma dan tipe token |
| **Payload** | Data (contoh: `{ userId: 1 }`) |
| **Signature** | Tanda tangan untuk memverifikasi keaslian token |

---

## Bagian A: Setup Auth Module

### Langkah 1: Generate Auth Resource

```bash
npx nest generate resource
```

Jawab pertanyaan:
1. **Nama:** `auth`
2. **Transport:** `REST API`
3. **CRUD entry points?** `No` ← Kita buat sendiri

### Langkah 2: Install & Konfigurasi Passport + JWT

#### 2.1 Install Dependencies

```bash
npm install --save @nestjs/passport passport @nestjs/jwt passport-jwt
npm install --save-dev @types/passport-jwt
```

| Package | Fungsi |
|---------|--------|
| `@nestjs/passport` | Wrapper Passport untuk NestJS |
| `passport` | Library authentication populer untuk Node.js |
| `@nestjs/jwt` | Wrapper JWT untuk NestJS |
| `passport-jwt` | Strategy JWT untuk Passport |

#### 2.2 Konfigurasi Auth Module

```typescript
// src/auth/auth.module.ts

import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { PrismaModule } from 'src/prisma/prisma.module';

// ⚠️ Di produksi, simpan secret di environment variable!
export const jwtSecret = 'zjP9h6ZI5LoSKCRj';

@Module({
  imports: [
    PrismaModule,
    PassportModule,
    JwtModule.register({
      secret: jwtSecret,
      signOptions: { expiresIn: '5m' },  // Token berlaku 5 menit
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```

> 💡 **Penjelasan:**
> - `PassportModule` → Mengaktifkan fitur Passport di NestJS
> - `JwtModule.register()` → Mengkonfigurasi JWT dengan secret key dan waktu expiry
> - `expiresIn: '5m'` → Token akan kadaluarsa setelah 5 menit. Bisa juga `'7d'`, `'24h'`, `'30s'`

> ⚠️ **Keamanan:** Jangan pernah hardcode secret key di kode! Gunakan `@nestjs/config` dan environment variable di produksi.

---

## Bagian B: Membuat Login Endpoint

### Langkah 3: Membuat Login DTO & Auth Entity

#### Login DTO

```typescript
// src/auth/dto/login.dto.ts

import { ApiProperty } from '@nestjs/swagger';
import { IsEmail, IsNotEmpty, IsString, MinLength } from 'class-validator';

export class LoginDto {
  @IsEmail()
  @IsNotEmpty()
  @ApiProperty()
  email: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(6)
  @ApiProperty()
  password: string;
}
```

#### Auth Entity (Response)

```typescript
// src/auth/entity/auth.entity.ts

import { ApiProperty } from '@nestjs/swagger';

export class AuthEntity {
  @ApiProperty()
  accessToken: string;  // ← JWT token yang akan dikembalikan
}
```

### Langkah 4: Implementasi Login di AuthService

```typescript
// src/auth/auth.service.ts

import {
  Injectable,
  NotFoundException,
  UnauthorizedException,
} from '@nestjs/common';
import { PrismaService } from './../prisma/prisma.service';
import { JwtService } from '@nestjs/jwt';
import { AuthEntity } from './entity/auth.entity';

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async login(email: string, password: string): Promise<AuthEntity> {
    // 🔍 Langkah 1: Cari user berdasarkan email
    const user = await this.prisma.user.findUnique({
      where: { email: email },
    });

    // ❌ Jika user tidak ditemukan
    if (!user) {
      throw new NotFoundException(`No user found for email: ${email}`);
    }

    // 🔐 Langkah 2: Verifikasi password
    const isPasswordValid = user.password === password;

    // ❌ Jika password salah
    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid password');
    }

    // ✅ Langkah 3: Generate JWT token
    return {
      accessToken: this.jwtService.sign({ userId: user.id }),
    };
  }
}
```

> 💡 **Alur Login:**
> 1. **Cari user** → Jika tidak ada → `404 Not Found`
> 2. **Cek password** → Jika salah → `401 Unauthorized`
> 3. **Buat token** → Token berisi `{ userId: 1 }` → Kirim ke client

> ⚠️ **Perhatian:** Saat ini kita masih membandingkan password dalam **plain text**. Ini akan diperbaiki di Bagian E menggunakan bcrypt.

### Langkah 5: Membuat Login Controller

```typescript
// src/auth/auth.controller.ts

import { Body, Controller, Post } from '@nestjs/common';
import { AuthService } from './auth.service';
import { ApiOkResponse, ApiTags } from '@nestjs/swagger';
import { AuthEntity } from './entity/auth.entity';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
@ApiTags('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('login')
  @ApiOkResponse({ type: AuthEntity })
  login(@Body() { email, password }: LoginDto) {
    return this.authService.login(email, password);
  }
}
```

#### Test Login

Kirim request ke `POST /auth/login`:

```json
{
  "email": "sabin@adams.com",
  "password": "password-sabin"
}
```

Response yang diharapkan:

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

---

## Bagian C: Melindungi Endpoint dengan JWT

### Langkah 6: Membuat JWT Strategy

JWT Strategy menentukan **bagaimana** token di-validasi.

```typescript
// src/auth/jwt.strategy.ts

import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { jwtSecret } from './auth.module';
import { UsersService } from 'src/users/users.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(private usersService: UsersService) {
    super({
      // 🔍 Ambil token dari header: Authorization: Bearer <token>
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      // 🔑 Gunakan secret yang sama untuk verifikasi
      secretOrKey: jwtSecret,
    });
  }

  // ✅ Method ini dipanggil SETELAH token berhasil diverifikasi
  async validate(payload: { userId: number }) {
    const user = await this.usersService.findOne(payload.userId);

    if (!user) {
      throw new UnauthorizedException();
    }

    return user;  // User akan tersedia di request.user
  }
}
```

> 💡 **Alur Validasi Token:**
> ```
> Request masuk → Extract token dari header
>       │
>       ▼
> Verifikasi signature dengan secret
>       │
>       ▼
> Decode payload: { userId: 1 }
>       │
>       ▼
> Panggil validate() → Cari user di database
>       │
>       ▼
> User ditemukan → Lanjutkan request ✅
> User tidak ada → 401 Unauthorized ❌
> ```

#### Update Auth Module (tambahkan JwtStrategy & UsersModule)

```typescript
// src/auth/auth.module.ts

import { Module } from '@nestjs/common';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { PassportModule } from '@nestjs/passport';
import { JwtModule } from '@nestjs/jwt';
import { PrismaModule } from 'src/prisma/prisma.module';
import { UsersModule } from 'src/users/users.module';        // ← BARU
import { JwtStrategy } from './jwt.strategy';                 // ← BARU

export const jwtSecret = 'zjP9h6ZI5LoSKCRj';

@Module({
  imports: [
    PrismaModule,
    PassportModule,
    JwtModule.register({
      secret: jwtSecret,
      signOptions: { expiresIn: '5m' },
    }),
    UsersModule,  // ← BARU: Agar bisa akses UsersService
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],  // ← BARU: Tambahkan JwtStrategy
})
export class AuthModule {}
```

#### Update Users Module (export UsersService)

```typescript
// src/users/users.module.ts

import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { PrismaModule } from 'src/prisma/prisma.module';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  imports: [PrismaModule],
  exports: [UsersService],  // ← BARU: Agar AuthModule bisa pakai UsersService
})
export class UsersModule {}
```

### Langkah 7: Membuat JWT Auth Guard

**Guard** menentukan apakah sebuah request **boleh dilanjutkan** atau **ditolak**.

```typescript
// src/auth/jwt-auth.guard.ts

import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

> 💡 **Penjelasan:** `AuthGuard('jwt')` akan otomatis menggunakan `JwtStrategy` yang kita buat sebelumnya.

### Langkah 8: Menerapkan Guard ke Endpoint

Tambahkan `@UseGuards(JwtAuthGuard)` ke endpoint yang ingin dilindungi:

```typescript
// src/users/users.controller.ts

import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  ParseIntPipe,
  UseGuards,  // ← Tambahkan
} from '@nestjs/common';
import { JwtAuthGuard } from 'src/auth/jwt-auth.guard';  // ← Tambahkan

@Controller('users')
@ApiTags('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  // 🔓 POST /users — TERBUKA (untuk registrasi)
  @Post()
  @ApiCreatedResponse({ type: UserEntity })
  async create(@Body() createUserDto: CreateUserDto) {
    return new UserEntity(await this.usersService.create(createUserDto));
  }

  // 🔒 GET /users — DILINDUNGI
  @Get()
  @UseGuards(JwtAuthGuard)
  @ApiOkResponse({ type: UserEntity, isArray: true })
  async findAll() {
    const users = await this.usersService.findAll();
    return users.map((user) => new UserEntity(user));
  }

  // 🔒 GET /users/:id — DILINDUNGI
  @Get(':id')
  @UseGuards(JwtAuthGuard)
  @ApiOkResponse({ type: UserEntity })
  async findOne(@Param('id', ParseIntPipe) id: number) {
    return new UserEntity(await this.usersService.findOne(id));
  }

  // 🔒 PATCH /users/:id — DILINDUNGI
  @Patch(':id')
  @UseGuards(JwtAuthGuard)
  @ApiCreatedResponse({ type: UserEntity })
  async update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return new UserEntity(await this.usersService.update(id, updateUserDto));
  }

  // 🔒 DELETE /users/:id — DILINDUNGI
  @Delete(':id')
  @UseGuards(JwtAuthGuard)
  @ApiOkResponse({ type: UserEntity })
  async remove(@Param('id', ParseIntPipe) id: number) {
    return new UserEntity(await this.usersService.remove(id));
  }
}
```

#### Hasil: Request Tanpa Token → 401

```json
// GET /users tanpa token
{
  "statusCode": 401,
  "message": "Unauthorized"
}
```

#### Endpoint yang Dilindungi

| Endpoint | Status | Alasan |
|----------|--------|--------|
| `POST /users` | 🔓 Terbuka | Untuk registrasi user baru |
| `GET /users` | 🔒 Dilindungi | Hanya user yang login |
| `GET /users/:id` | 🔒 Dilindungi | Hanya user yang login |
| `PATCH /users/:id` | 🔒 Dilindungi | Hanya user yang login |
| `DELETE /users/:id` | 🔒 Dilindungi | Hanya user yang login |
| `POST /auth/login` | 🔓 Terbuka | Untuk login |

---

## Bagian D: Integrasi Auth di Swagger

### Langkah 9: Konfigurasi Bearer Auth di Swagger

#### 9.1 Tambahkan `@ApiBearerAuth()` ke Endpoint yang Dilindungi

```typescript
// src/users/users.controller.ts

import {
  ApiBearerAuth,  // ← Tambahkan
  ApiCreatedResponse,
  ApiOkResponse,
  ApiTags,
} from '@nestjs/swagger';

@Controller('users')
@ApiTags('users')
export class UsersController {
  // ...

  @Get()
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()  // ← Tambahkan ke setiap endpoint yang dilindungi
  @ApiOkResponse({ type: UserEntity, isArray: true })
  async findAll() { /* ... */ }

  @Get(':id')
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()  // ← Tambahkan
  @ApiOkResponse({ type: UserEntity })
  async findOne() { /* ... */ }

  @Patch(':id')
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()  // ← Tambahkan
  @ApiCreatedResponse({ type: UserEntity })
  async update() { /* ... */ }

  @Delete(':id')
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()  // ← Tambahkan
  @ApiOkResponse({ type: UserEntity })
  async remove() { /* ... */ }
}
```

#### 9.2 Aktifkan Bearer Auth di Swagger Config

```typescript
// src/main.ts

const config = new DocumentBuilder()
  .setTitle('Median')
  .setDescription('The Median API description')
  .setVersion('0.1')
  .addBearerAuth()  // ← Tambahkan ini
  .build();
```

#### Cara Pakai di Swagger

1. **Login dulu** → `POST /auth/login` → Copy `accessToken`
2. **Klik tombol "Authorize"** 🔒 di bagian atas Swagger
3. **Paste token** → Klik "Authorize"
4. Sekarang semua request akan otomatis menyertakan token! ✅

---

## Bagian E: Hashing Password dengan Bcrypt

### ❌ Masalah: Password Tersimpan Plain Text!

```
Database saat ini:
┌──────────────────┬─────────────────┐
│ email            │ password        │
├──────────────────┼─────────────────┤
│ sabin@adams.com  │ password-sabin  │  ← 🚨 BAHAYA!
│ alex@ruheni.com  │ password-alex   │  ← 🚨 BAHAYA!
└──────────────────┴─────────────────┘

Yang seharusnya:
┌──────────────────┬──────────────────────────────────┐
│ email            │ password                         │
├──────────────────┼──────────────────────────────────┤
│ sabin@adams.com  │ $2b$10$XKQvtyb2Y.jciqh...       │  ← ✅ AMAN!
│ alex@ruheni.com  │ $2b$10$0tEfezrEd1a2g51...       │  ← ✅ AMAN!
└──────────────────┴──────────────────────────────────┘
```

### Apa itu Bcrypt?

**Bcrypt** adalah algoritma hashing yang dirancang khusus untuk password:

- 🔒 **One-way:** Hash tidak bisa di-reverse menjadi password asli
- 🧂 **Salting:** Menambahkan random string sebelum hashing (anti brute-force)
- ⏱️ **Cost factor:** Semakin tinggi = semakin lambat = semakin aman

### Langkah 10: Hash Password saat Create & Update User

#### 10.1 Install Bcrypt

```bash
npm install bcrypt
npm install --save-dev @types/bcrypt
```

#### 10.2 Update UsersService

```typescript
// src/users/users.service.ts

import { Injectable } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { PrismaService } from 'src/prisma/prisma.service';
import * as bcrypt from 'bcrypt';

export const roundsOfHashing = 10;

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  // CREATE — Hash password sebelum simpan
  async create(createUserDto: CreateUserDto) {
    const hashedPassword = await bcrypt.hash(
      createUserDto.password,
      roundsOfHashing,
    );
    createUserDto.password = hashedPassword;

    return this.prisma.user.create({ data: createUserDto });
  }

  findAll() {
    return this.prisma.user.findMany();
  }

  findOne(id: number) {
    return this.prisma.user.findUnique({ where: { id } });
  }

  // UPDATE — Hash password jika diubah
  async update(id: number, updateUserDto: UpdateUserDto) {
    if (updateUserDto.password) {
      updateUserDto.password = await bcrypt.hash(
        updateUserDto.password,
        roundsOfHashing,
      );
    }

    return this.prisma.user.update({
      where: { id },
      data: updateUserDto,
    });
  }

  remove(id: number) {
    return this.prisma.user.delete({ where: { id } });
  }
}
```

> 💡 **`roundsOfHashing = 10`** berarti bcrypt akan melakukan 2^10 = 1024 iterasi. Semakin tinggi angkanya, semakin aman tapi semakin lambat.

### Langkah 11: Update Seed Script

```typescript
// prisma/seed.ts

import { PrismaClient } from '@prisma/client';
import * as bcrypt from 'bcrypt';

const prisma = new PrismaClient();
const roundsOfHashing = 10;

async function main() {
  // ✅ Hash password sebelum menyimpan
  const passwordSabin = await bcrypt.hash('password-sabin', roundsOfHashing);
  const passwordAlex = await bcrypt.hash('password-alex', roundsOfHashing);

  const user1 = await prisma.user.upsert({
    where: { email: 'sabin@adams.com' },
    update: { password: passwordSabin },
    create: {
      email: 'sabin@adams.com',
      name: 'Sabin Adams',
      password: passwordSabin,  // ← Password yang sudah di-hash
    },
  });

  const user2 = await prisma.user.upsert({
    where: { email: 'alex@ruheni.com' },
    update: { password: passwordAlex },
    create: {
      email: 'alex@ruheni.com',
      name: 'Alex Ruheni',
      password: passwordAlex,  // ← Password yang sudah di-hash
    },
  });

  // ... (buat artikel seperti sebelumnya)

  console.log({ user1, user2 });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

Jalankan seed ulang:

```bash
npx prisma db seed
```

### Langkah 12: Update Login untuk Bcrypt

Sekarang password di database sudah di-hash, jadi login harus menggunakan `bcrypt.compare()`:

```typescript
// src/auth/auth.service.ts

import {
  Injectable,
  NotFoundException,
  UnauthorizedException,
} from '@nestjs/common';
import { PrismaService } from './../prisma/prisma.service';
import { JwtService } from '@nestjs/jwt';
import { AuthEntity } from './entity/auth.entity';
import * as bcrypt from 'bcrypt';  // ← Tambahkan

@Injectable()
export class AuthService {
  constructor(
    private prisma: PrismaService,
    private jwtService: JwtService,
  ) {}

  async login(email: string, password: string): Promise<AuthEntity> {
    const user = await this.prisma.user.findUnique({
      where: { email },
    });

    if (!user) {
      throw new NotFoundException(`No user found for email: ${email}`);
    }

    // ✅ Bandingkan password input dengan hash di database
    const isPasswordValid = await bcrypt.compare(password, user.password);

    if (!isPasswordValid) {
      throw new UnauthorizedException('Invalid password');
    }

    return {
      accessToken: this.jwtService.sign({ userId: user.id }),
    };
  }
}
```

> 💡 **`bcrypt.compare()`** secara otomatis:
> 1. Mengekstrak salt dari hash yang tersimpan
> 2. Hash password input dengan salt yang sama
> 3. Membandingkan kedua hash
> 4. Mengembalikan `true` jika cocok

---

## 💡 Alur Lengkap Authentication

```
=== LOGIN ===
Client                          Server
  │                               │
  │  POST /auth/login             │
  │  { email, password }          │
  │ ─────────────────────────────>│
  │                               │  1. Cari user by email
  │                               │  2. bcrypt.compare(password, hash)
  │                               │  3. Generate JWT { userId: 1 }
  │  { accessToken: "eyJ..." }    │
  │ <─────────────────────────────│
  │                               │

=== AKSES PROTECTED ENDPOINT ===
Client                          Server
  │                               │
  │  GET /users                   │
  │  Header: Authorization:       │
  │  Bearer eyJ...                │
  │ ─────────────────────────────>│
  │                               │  1. Extract token dari header
  │                               │  2. Verifikasi signature
  │                               │  3. Decode payload { userId: 1 }
  │                               │  4. Cari user di database
  │                               │  5. User ditemukan → ✅ Lanjutkan
  │  [{ id: 1, name: "..." }]    │
  │ <─────────────────────────────│
```

---

## File `main.ts` Final (Lengkap)

```typescript
// src/main.ts

import { HttpAdapterHost, NestFactory, Reflector } from '@nestjs/core';
import { AppModule } from './app.module';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { ClassSerializerInterceptor, ValidationPipe } from '@nestjs/common';
import { PrismaClientExceptionFilter } from './prisma-client-exception/prisma-client-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // ✅ Validasi input
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));

  // ✅ Serialisasi (sembunyikan password)
  app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));

  // ✅ Swagger dengan Bearer Auth
  const config = new DocumentBuilder()
    .setTitle('Median')
    .setDescription('The Median API description')
    .setVersion('0.1')
    .addBearerAuth()
    .build();
  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('api', app, document);

  // ✅ Exception Filter untuk Prisma
  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new PrismaClientExceptionFilter(httpAdapter));

  await app.listen(3000);
}
bootstrap();
```

---

## Ringkasan

| ✅ | Yang Sudah Dipelajari |
|----|----------------------|
| 1 | Memahami konsep JWT Authentication |
| 2 | Menginstal & mengkonfigurasi Passport + JWT |
| 3 | Membuat endpoint `POST /auth/login` |
| 4 | Membuat JWT Strategy untuk validasi token |
| 5 | Membuat Auth Guard untuk melindungi endpoint |
| 6 | Menerapkan `@UseGuards()` dan `@ApiBearerAuth()` |
| 7 | Mengintegrasikan Bearer Auth di Swagger |
| 8 | Hashing password dengan bcrypt |
| 9 | Update login untuk membandingkan hash |

### 🏁 Selamat!

Kamu telah menyelesaikan seluruh series! Sekarang kamu memiliki **REST API yang lengkap** dengan:

- ✅ CRUD Operations untuk Articles & Users
- ✅ Input Validation & Error Handling
- ✅ Relational Data (User ↔ Article)
- ✅ JWT Authentication
- ✅ Password Hashing
- ✅ Swagger Documentation

### 📚 Referensi Lanjutan

- [NestJS Documentation](https://docs.nestjs.com/)
- [Prisma Documentation](https://www.prisma.io/docs/)
- [Passport.js Documentation](https://www.passportjs.org/)
- [JWT.io](https://jwt.io/) — Untuk decode & debug JWT token

---

## 📝 Laporan Praktikum — Chapter 4

> **Instruksi:** Centang setiap langkah yang sudah selesai dikerjakan sebagai bukti laporan praktikum.
> Ubah `[ ]` menjadi `[x]` untuk menandai selesai.

### Part A: Setup Auth Module
- [x] Membuat resource Auth (`npx nest generate resource` → auth, REST API, No CRUD)
- [x] Menginstal dependensi Passport & JWT (`@nestjs/passport`, `passport`, `@nestjs/jwt`, `passport-jwt`)
- [x] Menginstal type definitions (`@types/passport-jwt`)
- [x] Mengonfigurasi `AuthModule` dengan `PassportModule` dan `JwtModule.register()`
- [x] Mengatur `jwtSecret` dan `expiresIn: '5m'`

### Part B: Login Endpoint
- [x] Membuat `LoginDto` dengan validasi (`@IsEmail`, `@IsNotEmpty`, `@MinLength`)
- [x] Membuat `AuthEntity` dengan field `accessToken`
- [x] Mengimplementasikan `AuthService.login()` — cari user, verifikasi password, generate token
- [x] Membuat `POST /auth/login` di `AuthController`
- [x] Menambahkan `@ApiTags('auth')` dan `@ApiOkResponse({ type: AuthEntity })`
- [x] Menguji login di Swagger — mengirim email & password yang benar
- [x] Menguji login — mengirim email yang tidak ada → melihat error 404
- [x] Menguji login — mengirim password yang salah → melihat error 401

### Part C: Melindungi Endpoint dengan JWT
- [x] Membuat `JwtStrategy` (extends `PassportStrategy`)
- [x] Mengonfigurasi `ExtractJwt.fromAuthHeaderAsBearerToken()`
- [x] Mengimplementasikan method `validate()` di `JwtStrategy`
- [x] Menambahkan `JwtStrategy` ke `providers` di `AuthModule`
- [x] Mengimpor `UsersModule` di `AuthModule`
- [x] Menambahkan `exports: [UsersService]` di `UsersModule`
- [x] Membuat `JwtAuthGuard` (extends `AuthGuard('jwt')`)
- [x] Menerapkan `@UseGuards(JwtAuthGuard)` ke `GET /users`
- [x] Menerapkan `@UseGuards(JwtAuthGuard)` ke `GET /users/:id`
- [x] Menerapkan `@UseGuards(JwtAuthGuard)` ke `PATCH /users/:id`
- [x] Menerapkan `@UseGuards(JwtAuthGuard)` ke `DELETE /users/:id`
- [x] Memverifikasi `POST /users` tetap terbuka (tanpa guard)
- [x] Menguji — mengakses `GET /users` tanpa token → melihat error 401

### Part D: Integrasi Auth di Swagger
- [x] Menambahkan `.addBearerAuth()` di `DocumentBuilder` (`main.ts`)
- [x] Menambahkan `@ApiBearerAuth()` ke setiap endpoint yang dilindungi
- [x] Menguji di Swagger — login → copy token → klik Authorize → paste token
- [x] Menguji — mengakses endpoint yang dilindungi setelah authorize → berhasil 200

### Part E: Hashing Password dengan Bcrypt
- [x] Menginstal `bcrypt` dan `@types/bcrypt`
- [x] Memperbarui `UsersService.create()` — hash password sebelum simpan
- [x] Memperbarui `UsersService.update()` — hash password jika diubah
- [x] Memperbarui `prisma/seed.ts` — gunakan `bcrypt.hash()` untuk password
- [x] Menjalankan seed ulang (`npx prisma db seed`)
- [x] Memperbarui `AuthService.login()` — gunakan `bcrypt.compare()`
- [x] Menguji login setelah bcrypt — memverifikasi login masih berfungsi
- [x] Menguji — membuat user baru dan memverifikasi password tersimpan sebagai hash

### Verifikasi Akhir
- [x] Login berhasil dan mendapatkan JWT token
- [x] Endpoint yang dilindungi menolak request tanpa token (401)
- [x] Endpoint yang dilindungi menerima request dengan token valid (200)
- [x] Password tersimpan sebagai hash di database (bukan plain text)
- [x] Swagger Bearer Auth berfungsi dengan benar

### ✅ Status Chapter 4
- [x] **SEMUA LANGKAH SELESAI** — Chapter 4 telah dikerjakan seluruhnya

| Item | Keterangan |
|------|------------|
| Nama | Nurul Insani Alimin |
| NIM | 105841108321 |
| Tanggal | 20 Februari 2026 |
| Tanda Tangan | NurulInsaniAlimin |
