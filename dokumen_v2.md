<div align="center">

# 💳 Payment Service Terpusat

### `payment.test` — Gateway Pembayaran untuk Ekosistem Ovisito

**DANA · Sanctum ClientApp · HMAC Webhook · Idempotent · Self-Healing**

![Versi](https://img.shields.io/badge/versi-2.0.0-0ea5e9?style=for-the-badge)
![Laravel](https://img.shields.io/badge/Laravel-10%20%7C%2011-FF2D20?style=for-the-badge&logo=laravel&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.2+-777BB4?style=for-the-badge&logo=php&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-8+-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Status](https://img.shields.io/badge/status-siap%20eksekusi-16a34a?style=for-the-badge)

*PT Balai Digital Teknologi · Tim Engineering Ovisito · 5 November 2026*

</div>

---

## 📌 Ringkasan Dokumen

| | |
|---|---|
| **Tujuan** | Membangun layanan pembayaran terpusat yang menangani DANA (dan gateway lain kelak) atas nama `ovisito.com` |
| **Versi** | 2.0.0 — migrasi dari API Key statis ke **Sanctum ClientApp** + **HMAC webhook**, konsisten dengan pola `ovisitomcp.test` |
| **Audiens** | Engineer backend yang mengimplementasikan `payment.test` dan sisi penerima di `ovisitomcp.test` |
| **Waktu implementasi** | ± 2 jam 45 menit (belum termasuk pengujian sandbox) |
| **Prasyarat pengetahuan** | Laravel, Sanctum, Eloquent, queue/scheduler, dasar HMAC & RSA |

### 🎯 Alur Bisnis

1. User memesan tiket di `ovisito.com/transport` → mendapat **kode booking**.
2. User memilih **Bayar DANA** → diarahkan ke `payment.test` → **DANA cashier**.
3. DANA mengirim **webhook** ke `payment.test` → diverifikasi → **callback HMAC** ke `ovisito.com` → **WhatsApp terkirim otomatis**.

> [!IMPORTANT]
> **Prinsip kunci:** `payment.test` tidak tahu soal booking, observer, atau WhatsApp. Ia hanya menerima request, membuat transaksi, memverifikasi webhook, dan mengirim callback. Seluruh *business logic* tetap di Ovisito.

---

## 📑 Daftar Isi

| # | Bagian | # | Bagian |
|---|---|---|---|
| 1 | [Arsitektur & Alur](#1--arsitektur--alur) | 10 | [Routing & Keamanan](#10--routing--keamanan) |
| 2 | [Persiapan Kredensial](#2--persiapan-kredensial--secrets) | 11 | [Sisi ovisitomcp.test](#11--sisi-ovisitomcptest) |
| 3 | [Struktur Folder](#3--struktur-folder) | 12 | [Scheduler & Reliability](#12--scheduler--reliability) |
| 4 | [Konfigurasi](#4--konfigurasi-env--config) | 13 | [Pengujian End-to-End](#13--pengujian-end-to-end) |
| 5 | [Migration Database](#5--migration-database) | 14 | [Deployment Checklist](#14--deployment-checklist) |
| 6 | [Model Eloquent](#6--model-eloquent) | 15 | [Menambah Gateway Baru](#15--menambah-gateway-baru) |
| 7 | [Contract & Gateway DANA](#7--contract--gateway-dana) | 16 | [Troubleshooting](#16--troubleshooting) |
| 8 | [Service Layer](#8--service-layer) | 17 | [Perubahan v1 → v2](#17--perubahan-v1--v2) |
| 9 | [HTTP Layer](#9--http-layer-controllers) | 18 | [Rencana Eksekusi](#18--rencana-eksekusi) |

---

## 1. 🏗️ Arsitektur & Alur

### 1.1 Diagram Komponen

```mermaid
flowchart TB
    subgraph OVI["🟦 ovisitomcp.test — Laravel (EXISTING)"]
        A1["TransportWebController<br/>→ PaymentClientService"]
        A2["TransportBookingObserver<br/>→ WhatsApp (Fonnte)"]
        A3["POST /api/internal/payment-callback<br/>(verifikasi HMAC)"]
    end

    subgraph PAY["🟩 payment.test — Laravel (BARU)"]
        B1["Sanctum ClientApp<br/>/api/v1/auth/*"]
        B2["PaymentController<br/>/api/v1/payment/*"]
        B3["WebhookController<br/>/api/webhook/dana"]
        B4["OvisitoCallbackService<br/>(HMAC outbound)"]
        B5[("payment_gateway<br/>client_apps · payment_transactions<br/>failed_callbacks")]
    end

    subgraph DANA["🟨 DANA Sandbox / Production"]
        C1["Cashier & Finish Notify"]
    end

    A1 -- "① login → Bearer token<br/>② POST /payment/create" --> B1
    B2 -- "③ createOrder" --> C1
    C1 -- "④ webhook (X-SIGNATURE RSA)" --> B3
    B3 --> B4
    B4 -- "⑤ callback (X-Payment-Signature)" --> A3
    A3 --> A2
    B1 & B2 & B3 & B4 --- B5
```

### 1.2 Diagram Urutan (Happy Path)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant O as ovisitomcp.test
    participant P as payment.test
    participant D as DANA

    U->>O: Pilih "Bayar DANA" (kode booking)
    O->>P: POST /v1/auth/login (cache token)
    O->>P: POST /v1/payment/create (Bearer)
    P->>D: createOrder
    D-->>P: acquirementId + checkoutUrl
    P-->>O: { checkout_url, external_id }
    O-->>U: Redirect ke checkout_url
    U->>D: Bayar di cashier
    D->>P: Webhook Finish Notify (X-SIGNATURE)
    P->>P: Verifikasi signature · set PAID (lock + idempotent)
    P->>O: POST /internal/payment-callback (HMAC)
    O->>O: Verifikasi HMAC · booking = PAID
    O-->>U: 📲 WhatsApp e-ticket (via Observer)
    D-->>U: Redirect ke /payment/return
```

### 1.3 Tiga Mekanisme Autentikasi

| Arah | Mekanisme | Header |
|---|---|---|
| **Ovisito → payment.test** | Sanctum Bearer (ClientApp) | `Authorization: Bearer {token}` |
| **DANA → payment.test** | Signature RSA | `X-SIGNATURE`, `X-PARTNER-ID`, `X-TIMESTAMP` |
| **payment.test → Ovisito** | HMAC webhook outbound | `X-Payment-Signature`, `X-Payment-Timestamp`, `X-Payment-Event`, `X-Payment-Delivery-Id` |

### 1.4 Status Transaksi

```mermaid
stateDiagram-v2
    [*] --> PENDING
    PENDING --> PAID: webhook / polling sukses
    PENDING --> EXPIRED: lewat expires_at
    PENDING --> FAILED: ditolak / dibatalkan
    PAID --> [*]
    EXPIRED --> [*]
    FAILED --> [*]
```

---

## 2. 🔐 Persiapan Kredensial & Secrets

### 2.1 Kredensial DANA Sandbox

Login ke `dashboard.dana.id/sandbox`, lalu:

1. **Buat Merchant** → catat `merchantId`, `clientId`, `clientSecret`.
2. **Generate RSA key pair** → simpan *private key*, unggah *public key*.
3. **Unduh DANA Public Key** (untuk verifikasi webhook).

### 2.2 Generate Secret Antar-Service

```bash
# 1) Client secret ClientApp (ovisito → payment)
php -r "echo 'OVISITO_CLIENT_SECRET=' . bin2hex(random_bytes(32)) . PHP_EOL;"

# 2) Webhook secret HMAC (payment → ovisito)
php -r "echo 'OVISITO_WEBHOOK_SECRET=' . bin2hex(random_bytes(32)) . PHP_EOL;"
```

| Variabel | Di `payment.test` | Di `ovisitomcp.test` |
|---|---|---|
| Client ID | `OVISITO_CLIENT_ID` = `ovisito-core` | `PAYMENT_CLIENT_ID` = `ovisito-core` |
| Client Secret | `OVISITO_CLIENT_SECRET` | `PAYMENT_CLIENT_SECRET` |
| Webhook Secret | `OVISITO_WEBHOOK_SECRET` | `PAYMENT_WEBHOOK_SECRET` |

> [!WARNING]
> Secret hanya disimpan di `.env`. Jangan di-commit, jangan ditempel di chat/tiket. Jika pernah terekspos, **rotasi segera**.

### 2.3 Domain Publik untuk Webhook (Lokal)

```bash
ngrok http 8000      # → https://xxxx.ngrok.io
```

Gunakan URL HTTPS ini untuk `DANA_ORDER_NOTIFY_URL` dan `DANA_PAY_RETURN_URL`.

---

## 3. 📂 Struktur Folder

```text
payment.test/
├── app/
│   ├── Console/Commands/
│   │   ├── RetryFailedCallbacks.php
│   │   ├── ExpirePendingTransactions.php
│   │   └── PollPendingTransactions.php
│   ├── Contracts/PaymentGatewayInterface.php
│   ├── Http/Controllers/
│   │   ├── Api/
│   │   │   ├── V1/{AuthController.php, PaymentController.php}
│   │   │   └── WebhookController.php
│   │   └── PaymentReturnController.php
│   ├── Models/{ClientApp.php, PaymentTransaction.php}
│   ├── Providers/AppServiceProvider.php        ← rate limiter
│   └── Services/
│       ├── Gateways/DanaGateway.php
│       ├── OvisitoCallbackService.php
│       └── PaymentManager.php
├── config/{dana.php, services.php}
├── database/
│   ├── migrations/
│   │   ├── 2026_11_05_000000_create_client_apps_table.php
│   │   ├── 2026_11_05_000001_create_payment_transactions_table.php
│   │   └── (auto) personal_access_tokens
│   └── seeders/ClientAppSeeder.php
├── resources/views/payment/return.blade.php
└── routes/{api.php, web.php, console.php}
```

---

## 4. ⚙️ Konfigurasi `.env` & `config`

### 4.1 `payment.test/.env`

```env
APP_NAME="Ovisito Payment"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://payment.test
APP_KEY=                      # php artisan key:generate

# ─── Database ─────────────────────────────────────────
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=payment_gateway
DB_USERNAME=root
DB_PASSWORD=

# ─── DANA ─────────────────────────────────────────────
DANA_ENV=sandbox
DANA_ACTIVE=true
DANA_API_URL=https://api-sandbox.saas.dana.id
DANA_WEB_URL=https://m.sandbox.dana.id
DANA_MARCHANT_ID=<merchant-id>        # "MARCHANT" (typo) disengaja — mengikuti package
DANA_CLIENT_ID=<client-id>
DANA_CLIENT_SECRET=<client-secret>
DANA_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
DANA_PUB_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"
DANA_ORDER_NOTIFY_URL=https://xxxx.ngrok.io/api/webhook/dana
DANA_PAY_RETURN_URL=https://xxxx.ngrok.io/payment/return

# Production: true. Sandbox awal / uji curl lokal: boleh false.
PAYMENT_REQUIRE_SIGNATURE=true

# ─── ClientApp (Ovisito sebagai klien) ────────────────
OVISITO_CLIENT_ID=ovisito-core
OVISITO_CLIENT_SECRET=<hex-64>

# ─── Callback ke Ovisito ──────────────────────────────
OVISITO_CALLBACK_URL=https://ovisitomcp.test
OVISITO_CALLBACK_PATH=/api/internal/payment-callback
OVISITO_WEBHOOK_SECRET=<hex-64>
```

### 4.2 `payment.test/config/services.php`

```php
'ovisito' => [
    'client_id'      => env('OVISITO_CLIENT_ID'),
    'client_secret'  => env('OVISITO_CLIENT_SECRET'),
    'callback_url'   => env('OVISITO_CALLBACK_URL', 'http://ovisitomcp.test'),
    'callback_path'  => env('OVISITO_CALLBACK_PATH', '/api/internal/payment-callback'),
    'webhook_secret' => env('OVISITO_WEBHOOK_SECRET'),
],

'payment' => [
    'require_webhook_signature' => env('PAYMENT_REQUIRE_SIGNATURE', true),
],
```

### 4.3 `payment.test/config/dana.php` (verifikasi)

```php
"merchant_id"     => env("DANA_MARCHANT_ID", ""),   // typo MARCHANT disengaja
"ssh_public_key"  => env("DANA_PUB_KEY", ""),
"ssh_private_key" => env("DANA_PRIVATE_KEY", ""),
```

### 4.4 `ovisitomcp.test` — `.env` & `config/services.php`

```env
PAYMENT_BASE_URL=https://payment.ovisito.com/api/v1
PAYMENT_CLIENT_ID=ovisito-core
PAYMENT_CLIENT_SECRET=<sama dengan OVISITO_CLIENT_SECRET>
PAYMENT_WEBHOOK_SECRET=<sama dengan OVISITO_WEBHOOK_SECRET>
```

```php
'payment' => [
    'base_url'       => env('PAYMENT_BASE_URL'),
    'client_id'      => env('PAYMENT_CLIENT_ID'),
    'client_secret'  => env('PAYMENT_CLIENT_SECRET'),
    'webhook_secret' => env('PAYMENT_WEBHOOK_SECRET'),
],
```

### 4.5 Rate Limiter per Client

Kolom `rate_limit_per_minute` pada `client_apps` dipakai di sini. Tambahkan di `app/Providers/AppServiceProvider.php`:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot(): void
{
    RateLimiter::for('payment-api', function (Request $request) {
        $client = $request->user();
        return Limit::perMinute($client?->rate_limit_per_minute ?? 60)
            ->by($client?->id ?: $request->ip());
    });
}
```

---

## 5. 🗄️ Migration Database

### 5.1 Install Sanctum

```bash
cd /path/to/payment.test
composer require laravel/sanctum
php artisan install:api        # membuat migration personal_access_tokens
```

### 5.2 `client_apps`

`database/migrations/2026_11_05_000000_create_client_apps_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('client_apps', function (Blueprint $table) {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->string('name', 100);
            $table->string('client_id', 100)->unique();
            $table->string('client_secret', 255);          // di-hash via cast
            $table->string('platform', 20)->default('server');
            $table->boolean('is_active')->default(true);
            $table->unsignedInteger('rate_limit_per_minute')->default(120);
            $table->timestamp('last_used_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('client_apps');
    }
};
```

### 5.3 `payment_transactions` & `failed_callbacks`

`database/migrations/2026_11_05_000001_create_payment_transactions_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('payment_transactions', function (Blueprint $table) {
            $table->id();
            $table->uuid('uuid')->unique();

            $table->string('reference_id', 100)->index();    // booking_code
            $table->string('external_id', 100)->nullable(); // acquirementId DANA

            $table->string('provider', 30)->index();
            $table->decimal('amount', 15, 2);
            $table->char('currency', 3)->default('IDR');
            $table->string('status', 20)->default('PENDING')->index();
            // PENDING | PAID | EXPIRED | FAILED | CANCELLED

            $table->string('checkout_url', 500)->nullable();
            $table->timestamp('expires_at')->nullable();

            $table->json('metadata')->nullable();
            $table->timestamp('paid_at')->nullable();
            $table->timestamp('ovisito_synced_at')->nullable()
                ->comment('null = callback belum berhasil terkirim');
            $table->timestamps();

            $table->unique(['provider', 'external_id']);
            $table->index(['provider', 'status']);
            $table->index(['status', 'expires_at']);
            $table->index(['ovisito_synced_at', 'paid_at']);
        });

        Schema::create('failed_callbacks', function (Blueprint $table) {
            $table->id();
            $table->uuid('transaction_uuid')->index();
            $table->text('payload');
            $table->integer('attempts')->default(0);
            $table->text('last_error')->nullable();
            $table->timestamp('next_attempt_at')->nullable();
            $table->timestamp('resolved_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('failed_callbacks');
        Schema::dropIfExists('payment_transactions');
    }
};
```

### 5.4 Jalankan

```bash
mysql -u root -e "CREATE DATABASE payment_gateway CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
php artisan migrate
php artisan db:seed --class=ClientAppSeeder
```

---

## 6. 🧩 Model Eloquent

### 6.1 `app/Models/ClientApp.php`

```php
<?php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Support\Str;
use Laravel\Sanctum\HasApiTokens;

class ClientApp extends Authenticatable
{
    use HasApiTokens;

    protected $fillable = [
        'uuid', 'name', 'client_id', 'client_secret',
        'platform', 'is_active', 'rate_limit_per_minute', 'last_used_at',
    ];

    protected $hidden = ['client_secret'];

    protected $casts = [
        'client_secret' => 'hashed',
        'is_active'     => 'boolean',
        'last_used_at'  => 'datetime',
    ];

    protected static function booted(): void
    {
        static::creating(function (self $m) {
            if (empty($m->uuid)) {
                $m->uuid = (string) Str::uuid();
            }
        });
    }
}
```

### 6.2 `app/Models/PaymentTransaction.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class PaymentTransaction extends Model
{
    protected $table = 'payment_transactions';

    protected $fillable = [
        'uuid', 'reference_id', 'external_id', 'provider',
        'amount', 'currency', 'status',
        'checkout_url', 'expires_at', 'metadata',
        'paid_at', 'ovisito_synced_at',
    ];

    protected $casts = [
        'amount'            => 'decimal:2',
        'metadata'          => 'array',
        'expires_at'        => 'datetime',
        'paid_at'           => 'datetime',
        'ovisito_synced_at' => 'datetime',
    ];

    protected static function booted(): void
    {
        static::creating(function (self $m) {
            if (empty($m->uuid)) {
                $m->uuid = (string) Str::uuid();
            }
        });
    }

    public function isPending(): bool { return $this->status === 'PENDING'; }
    public function isPaid(): bool    { return $this->status === 'PAID'; }

    /** Final (PAID/EXPIRED/FAILED) tetapi Ovisito belum menerima callback. */
    public function needsCallback(): bool
    {
        return $this->ovisito_synced_at === null
            && in_array($this->status, ['PAID', 'EXPIRED', 'FAILED'], true);
    }
}
```

### 6.3 `database/seeders/ClientAppSeeder.php`

```php
<?php

namespace Database\Seeders;

use App\Models\ClientApp;
use Illuminate\Database\Seeder;

class ClientAppSeeder extends Seeder
{
    public function run(): void
    {
        $secret = env('OVISITO_CLIENT_SECRET');
        if (empty($secret)) {
            $this->command?->error('OVISITO_CLIENT_SECRET kosong — seeding dibatalkan.');
            return;
        }

        ClientApp::updateOrCreate(
            ['client_id' => env('OVISITO_CLIENT_ID', 'ovisito-core')],
            [
                'name'                  => 'Ovisito Core',
                'client_secret'         => $secret,      // di-hash otomatis oleh cast
                'platform'              => 'server',
                'is_active'             => true,
                'rate_limit_per_minute' => 120,
            ]
        );
    }
}
```

---

## 7. 🔌 Contract & Gateway DANA

### 7.1 `app/Contracts/PaymentGatewayInterface.php`

```php
<?php

namespace App\Contracts;

use Illuminate\Http\Request;

interface PaymentGatewayInterface
{
    public function getName(): string;

    /**
     * @param  array $data ['reference_id','amount','return_url','customer']
     * @return array ['external_id','checkout_url','expires_at']
     */
    public function createTransaction(array $data): array;

    /**
     * @return array ['status'=>'paid'|'expired'|'failed'|'unknown',
     *                'reference_id','external_id','amount','raw']
     * @throws \Exception jika signature tidak valid
     */
    public function handleWebhook(Request $request): array;
}
```

### 7.2 `app/Services/Gateways/DanaGateway.php`

```php
<?php

namespace App\Services\Gateways;

use App\Contracts\PaymentGatewayInterface;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class DanaGateway implements PaymentGatewayInterface
{
    private string $danaPublicKey;
    private int    $expiredAfter;
    private bool   $strictSignature;

    public function __construct()
    {
        $this->danaPublicKey   = (string) config('dana.ssh_public_key');
        $this->expiredAfter    = (int) config('dana.expired_after', 60);
        $this->strictSignature = (bool) config('services.payment.require_webhook_signature', true);
    }

    public function getName(): string
    {
        return 'dana';
    }

    // ── Create order ───────────────────────────────────────────────
    public function createTransaction(array $data): array
    {
        $expiry    = now()->addMinutes($this->expiredAfter);
        $returnUrl = $data['return_url'] ?? config('dana.pay_return_url');
        $returnUrl .= (str_contains($returnUrl, '?') ? '&' : '?')
                    . 'reference_id=' . urlencode($data['reference_id']);

        $payload = [
            'order' => [
                'orderTitle'        => 'Pembayaran ' . $data['reference_id'],
                'orderAmount'       => [
                    'currency' => 'IDR',
                    'value'    => (string) $data['amount'],     // verifikasi satuan — §16
                ],
                'merchantTransId'   => $data['reference_id'],
                'merchantTransType' => 'PURCHASE',
                'orderMemo'         => 'Pembayaran untuk ' . $data['reference_id'],
            ],
            'urlParams' => [
                ['url' => config('dana.order_notify_url'), 'type' => 'NOTIFICATION'],
                ['url' => $returnUrl,                      'type' => 'RETURN'],
            ],
            'expiryPeriod' => $expiry->format('Y-m-d\TH:i:sP'),
        ];

        $response = \Otnansirk\Dana\Facades\DANAPay::createOrder($payload);

        Log::info('[DANA] createOrder', [
            'reference_id' => $data['reference_id'],
            'response'     => $response,
        ]);

        $body = $response['response']['body'] ?? [];
        if (($body['resultInfo']['resultStatus'] ?? '') !== 'S') {
            throw new \RuntimeException(
                'DANA createOrder gagal: ' . json_encode($body['resultInfo'] ?? [])
            );
        }

        return [
            'external_id'  => $body['acquirementId'],
            'checkout_url' => $body['checkoutUrl'],
            'expires_at'   => $expiry,
        ];
    }

    // ── Webhook ────────────────────────────────────────────────────
    public function handleWebhook(Request $request): array
    {
        $signature = (string) $request->header('X-SIGNATURE', '');
        $rawBody   = $request->getContent();

        // Strict (default): signature WAJIB valid.
        // Non-strict (hanya sandbox awal): divalidasi bila header ada.
        if ($this->strictSignature || $signature !== '') {
            if (!$this->verifySignature($rawBody, $signature)) {
                Log::warning('[DANA] Invalid webhook signature');
                throw new \RuntimeException('Invalid signature');
            }
        }

        $body       = $request->input('response.body', $request->all());
        $danaStatus = strtoupper((string) ($body['latestTransactionStatus'] ?? ''));

        $status = match ($danaStatus) {
            'SUCCESS', 'PAID', 'SUCCESSFUL' => 'paid',
            'EXPIRED'                       => 'expired',
            'FAILED', 'CANCELLED', 'DENIED' => 'failed',
            default                         => 'unknown',
        };

        return [
            'status'       => $status,
            'reference_id' => $body['merchantTransId'] ?? null,
            'external_id'  => $body['acquirementId'] ?? null,
            'amount'       => (float) ($body['orderAmount']['value'] ?? 0),
            'raw'          => $body,
        ];
    }

    // ── Query (fallback webhook hilang) ────────────────────────────
    public function queryTransaction(string $acquirementId): array
    {
        $response   = \Otnansirk\Dana\Facades\DANAPay::queryOrder($acquirementId);
        $body       = $response['response']['body'] ?? [];
        $danaStatus = strtoupper((string) ($body['latestTransactionStatus'] ?? ''));

        $status = match ($danaStatus) {
            'SUCCESS', 'PAID', 'SUCCESSFUL' => 'paid',
            'EXPIRED'                       => 'expired',
            'FAILED', 'CANCELLED', 'DENIED' => 'failed',
            default                         => 'pending',
        };

        return [
            'status'      => $status,
            'external_id' => $body['acquirementId'] ?? $acquirementId,
            'amount'      => (float) ($body['orderAmount']['value'] ?? 0),
            'raw'         => $body,
        ];
    }

    // ── Verifikasi signature RSA ───────────────────────────────────
    private function verifySignature(string $rawBody, string $signature): bool
    {
        if ($this->danaPublicKey === '') {
            if ($this->strictSignature) {
                Log::error('[DANA] DANA_PUB_KEY kosong pada mode strict — webhook ditolak');
                return false;
            }
            Log::warning('[DANA] DANA_PUB_KEY kosong — bypass (non-strict)');
            return true;
        }

        if ($signature === '') {
            return false;
        }

        return openssl_verify(
            $rawBody,
            base64_decode($signature),
            $this->danaPublicKey,
            OPENSSL_ALGO_SHA256
        ) === 1;
    }
}
```

---

## 8. 🛠️ Service Layer

### 8.1 `app/Services/PaymentManager.php`

```php
<?php

namespace App\Services;

use App\Contracts\PaymentGatewayInterface;
use App\Services\Gateways\DanaGateway;
use InvalidArgumentException;

class PaymentManager
{
    public function driver(string $provider): PaymentGatewayInterface
    {
        return match (strtolower($provider)) {
            'dana' => new DanaGateway(),
            // 'flip'  => new FlipGateway(),
            // 'gopay' => new GopayGateway(),
            default => throw new InvalidArgumentException("Provider '{$provider}' tidak didukung."),
        };
    }
}
```

### 8.2 `app/Services/OvisitoCallbackService.php`

Perbaikan dari draf sebelumnya:

- **ID event deterministik** (`evt_{uuid transaksi}_{status}`) → retry tidak membuat event baru, sehingga idempotency di Ovisito bekerja.
- **Signature mencakup timestamp**: `HMAC(timestamp + "." + body)` → mencegah replay.
- **`$recordFailure`** → command retry tidak menggandakan baris `failed_callbacks`.

```php
<?php

namespace App\Services;

use App\Models\PaymentTransaction;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Str;

class OvisitoCallbackService
{
    /**
     * @param bool $recordFailure true = catat ke failed_callbacks bila gagal
     *                            (false saat dipanggil dari command retry)
     */
    public function sendResult(PaymentTransaction $tx, string $status, bool $recordFailure = true): bool
    {
        $eventName = match ($status) {
            'paid'    => 'payment.paid',
            'expired' => 'payment.expired',
            'failed'  => 'payment.failed',
            default   => 'payment.updated',
        };

        $payload = [
            'id'         => 'evt_' . $tx->uuid . '_' . $status,   // deterministik
            'type'       => $eventName,
            'created_at' => now()->toIso8601String(),
            'data'       => [
                'reference_id' => $tx->reference_id,
                'external_id'  => $tx->external_id,
                'provider'     => $tx->provider,
                'amount'       => (float) $tx->amount,
                'status'       => $status,
                'paid_at'      => $tx->paid_at?->toIso8601String(),
            ],
        ];

        $rawBody    = json_encode($payload, JSON_UNESCAPED_SLASHES);
        $secret     = (string) config('services.ovisito.webhook_secret');
        $timestamp  = now()->timestamp;
        $signature  = 'sha256=' . hash_hmac('sha256', $timestamp . '.' . $rawBody, $secret);
        $deliveryId = (string) Str::uuid();                          // unik per percobaan

        try {
            $response = Http::withHeaders([
                    'X-Payment-Signature'   => $signature,
                    'X-Payment-Timestamp'   => (string) $timestamp,
                    'X-Payment-Event'       => $eventName,
                    'X-Payment-Delivery-Id' => $deliveryId,
                ])
                ->withBody($rawBody, 'application/json')
                ->timeout(10)
                ->post(
                    config('services.ovisito.callback_url')
                    . config('services.ovisito.callback_path')
                );

            if ($response->successful()) {
                $tx->update(['ovisito_synced_at' => now()]);
                Log::info('[Callback] OK', [
                    'reference_id' => $tx->reference_id,
                    'event'        => $eventName,
                ]);
                return true;
            }

            Log::warning('[Callback] non-2xx', [
                'reference_id' => $tx->reference_id,
                'http_status'  => $response->status(),
                'body'         => substr($response->body(), 0, 500),
            ]);
            if ($recordFailure) {
                $this->recordFailure($tx, $payload, $response->status() . ': ' . substr($response->body(), 0, 500));
            }
            return false;

        } catch (\Throwable $e) {
            Log::error('[Callback] exception', [
                'reference_id' => $tx->reference_id,
                'error'        => $e->getMessage(),
            ]);
            if ($recordFailure) {
                $this->recordFailure($tx, $payload, $e->getMessage());
            }
            return false;
        }
    }

    private function recordFailure(PaymentTransaction $tx, array $payload, string $error): void
    {
        DB::table('failed_callbacks')->insert([
            'transaction_uuid' => $tx->uuid,
            'payload'          => json_encode($payload),
            'attempts'         => 0,
            'last_error'       => $error,
            'next_attempt_at'  => now()->addMinutes(2),
            'created_at'       => now(),
            'updated_at'       => now(),
        ]);
    }
}
```

---

## 9. 🌐 HTTP Layer (Controllers)

### 9.1 `Api/V1/AuthController.php`

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Models\ClientApp;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{
    public function login(Request $request)
    {
        $v = $request->validate([
            'client_id'     => 'required|string',
            'client_secret' => 'required|string',
            'device_name'   => 'nullable|string|max:100',
        ]);
        $device = $v['device_name'] ?? 'default';

        $client = ClientApp::where('client_id', $v['client_id'])->first();

        if (!$client || !Hash::check($v['client_secret'], $client->client_secret)) {
            return response()->json([
                'success' => false,
                'error'   => ['code' => 'INVALID_CREDENTIALS', 'message' => 'Client ID atau secret tidak valid.'],
            ], 401);
        }

        if (!$client->is_active) {
            return response()->json([
                'success' => false,
                'error'   => ['code' => 'CLIENT_INACTIVE', 'message' => 'Client tidak aktif.'],
            ], 403);
        }

        $client->tokens()->where('name', $device)->delete();

        $expiresAt = now()->addDays(30);
        $token     = $client->createToken($device, ['payment'], $expiresAt)->plainTextToken;

        $client->update(['last_used_at' => now()]);

        return response()->json([
            'success' => true,
            'data'    => [
                'access_token' => $token,
                'token_type'   => 'Bearer',
                'expires_at'   => $expiresAt->toIso8601String(),
                'client'       => [
                    'id'        => $client->id,
                    'name'      => $client->name,
                    'client_id' => $client->client_id,
                    'platform'  => $client->platform,
                ],
            ],
        ]);
    }

    public function me(Request $request)
    {
        $c = $request->user();
        return response()->json([
            'success' => true,
            'data'    => ['id' => $c->id, 'name' => $c->name, 'client_id' => $c->client_id, 'platform' => $c->platform],
        ]);
    }

    public function refresh(Request $request)
    {
        $client  = $request->user();
        $oldName = $client->currentAccessToken()->name;
        $client->currentAccessToken()->delete();

        $expiresAt = now()->addDays(30);
        $token     = $client->createToken($oldName, ['payment'], $expiresAt)->plainTextToken;

        return response()->json([
            'success' => true,
            'data'    => ['access_token' => $token, 'expires_at' => $expiresAt->toIso8601String()],
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();
        return response()->json(['success' => true, 'data' => ['message' => 'Logout']]);
    }
}
```

> [!NOTE]
> `device_name` kini **opsional** (default `default`). Pada draf sebelumnya field ini `required`, padahal contoh `curl` di bagian pengujian tidak mengirimnya sehingga login selalu 422.

### 9.2 `Api/V1/PaymentController.php`

```php
<?php

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Models\PaymentTransaction;
use App\Services\PaymentManager;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class PaymentController extends Controller
{
    public function __construct(private readonly PaymentManager $manager) {}

    public function create(Request $request)
    {
        $validated = $request->validate([
            'reference_id'   => 'required|string|max:100',
            'amount'         => 'required|numeric|min:1',
            'provider'       => 'required|string|in:dana,flip,bsi,gopay',
            'return_url'     => 'required|url',
            'customer'       => 'nullable|array',
            'customer.name'  => 'nullable|string|max:100',
            'customer.phone' => 'nullable|string|max:30',
            'customer.email' => 'nullable|email|max:150',
        ]);

        // Idempotency: pakai PENDING yang belum kedaluwarsa bila ada
        $existing = PaymentTransaction::where('reference_id', $validated['reference_id'])
            ->where('provider', $validated['provider'])
            ->where('status', 'PENDING')
            ->where('expires_at', '>', now())
            ->latest()
            ->first();

        if ($existing) {
            return response()->json(['success' => true, 'data' => $this->payload($existing, true)]);
        }

        try {
            $result = $this->manager->driver($validated['provider'])->createTransaction([
                'reference_id' => $validated['reference_id'],
                'amount'       => (float) $validated['amount'],
                'return_url'   => $validated['return_url'],
                'customer'     => $validated['customer'] ?? [],
            ]);
        } catch (\Throwable $e) {
            Log::error('[Payment] createTransaction failed', [
                'provider' => $validated['provider'],
                'error'    => $e->getMessage(),
            ]);

            return response()->json([
                'success' => false,
                'error'   => [
                    'code'    => 'GATEWAY_ERROR',
                    // jangan bocorkan detail internal di production
                    'message' => config('app.debug') ? $e->getMessage() : 'Gagal membuat transaksi.',
                ],
            ], 502);
        }

        $tx = PaymentTransaction::create([
            'reference_id' => $validated['reference_id'],
            'external_id'  => $result['external_id'],
            'provider'     => $validated['provider'],
            'amount'       => $validated['amount'],
            'currency'     => 'IDR',
            'status'       => 'PENDING',
            'checkout_url' => $result['checkout_url'],
            'expires_at'   => $result['expires_at'],
            'metadata'     => [
                'customer'   => $validated['customer'] ?? [],
                'return_url' => $validated['return_url'],
                'client_id'  => $request->user()->client_id,
            ],
        ]);

        return response()->json(['success' => true, 'data' => $this->payload($tx, false)]);
    }

    public function status(string $referenceId)
    {
        $tx = PaymentTransaction::where('reference_id', $referenceId)->latest()->first();

        if (!$tx) {
            return response()->json([
                'success' => false,
                'error'   => ['code' => 'NOT_FOUND', 'message' => 'Transaction not found.'],
            ], 404);
        }

        return response()->json([
            'success' => true,
            'data'    => [
                'reference_id' => $tx->reference_id,
                'external_id'  => $tx->external_id,
                'provider'     => $tx->provider,
                'status'       => $tx->status,
                'amount'       => (float) $tx->amount,
                'paid_at'      => $tx->paid_at?->toIso8601String(),
                'synced'       => $tx->ovisito_synced_at !== null,
            ],
        ]);
    }

    private function payload(PaymentTransaction $tx, bool $reused): array
    {
        return [
            'transaction_id' => $tx->uuid,
            'external_id'    => $tx->external_id,
            'checkout_url'   => $tx->checkout_url,
            'provider'       => $tx->provider,
            'expires_at'     => $tx->expires_at?->toIso8601String(),
            'reused'         => $reused,
        ];
    }
}
```

### 9.3 `Api/WebhookController.php`

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\PaymentTransaction;
use App\Services\OvisitoCallbackService;
use App\Services\PaymentManager;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Str;

class WebhookController extends Controller
{
    public function __construct(
        private readonly PaymentManager $manager,
        private readonly OvisitoCallbackService $callback,
    ) {}

    public function dana(Request $request)
    {
        try {
            $result = $this->manager->driver('dana')->handleWebhook($request);
        } catch (\Throwable $e) {
            Log::error('[Webhook] DANA handle failed', ['error' => $e->getMessage()]);
            return $this->errorResponse('INVALID_REQUEST', 401);
        }

        $externalId = $result['external_id'];
        $status     = $result['status'];

        if (!$externalId) {
            return $this->successResponse();
        }

        $tx = PaymentTransaction::where('external_id', $externalId)
            ->where('provider', 'dana')
            ->first();

        if (!$tx) {
            Log::warning('[Webhook] DANA transaksi tidak dikenal', ['external_id' => $externalId]);
            return $this->successResponse();
        }

        // Idempotency: sudah final → cukup pastikan callback tersinkron
        if (!$tx->isPending()) {
            if ($tx->needsCallback()) {
                $this->callback->sendResult($tx, strtolower($tx->status));
            }
            return $this->successResponse();
        }

        DB::transaction(function () use ($tx, $status, $result) {
            $fresh = PaymentTransaction::where('id', $tx->id)->lockForUpdate()->first();

            if (!$fresh->isPending()) {
                return;     // sudah diproses request paralel
            }

            $meta = array_merge($fresh->metadata ?? [], ['webhook_raw' => $result['raw']]);

            if ($status === 'paid') {
                $fresh->update(['status' => 'PAID', 'paid_at' => now(), 'metadata' => $meta]);
            } elseif (in_array($status, ['expired', 'failed'], true)) {
                $fresh->update(['status' => strtoupper($status), 'metadata' => $meta]);
            }
        });

        // Callback DI LUAR transaksi DB agar lock tidak tertahan
        $fresh = $tx->fresh();
        if ($fresh->needsCallback()) {
            $this->callback->sendResult($fresh, strtolower($fresh->status));
        }

        return $this->successResponse();
    }

    private function successResponse()
    {
        return response()->json([
            'response' => [
                'head' => [
                    'version'  => config('dana.version', '2.0'),
                    'function' => 'dana.notify.finish',
                    'reqTime'  => now()->format('Y-m-d\TH:i:sP'),
                    'reqMsgId' => (string) Str::uuid(),
                ],
                'body' => [
                    'resultInfo' => ['resultStatus' => 'S', 'resultCode' => 'SUCCESS', 'resultMsg' => 'OK'],
                ],
            ],
        ], 200);
    }

    private function errorResponse(string $code, int $httpStatus)
    {
        return response()->json([
            'response' => [
                'head' => ['version' => config('dana.version', '2.0')],
                'body' => [
                    'resultInfo' => ['resultStatus' => 'F', 'resultCode' => $code, 'resultMsg' => 'Error'],
                ],
            ],
        ], $httpStatus);
    }
}
```

### 9.4 `PaymentReturnController.php`

> [!NOTE]
> Halaman return **hanya menampilkan status** dan bukan sumber kebenaran. Sumber kebenaran adalah webhook DANA atau query status.

```php
<?php

namespace App\Http\Controllers;

use App\Models\PaymentTransaction;
use Illuminate\Http\Request;

class PaymentReturnController extends Controller
{
    public function __invoke(Request $request)
    {
        $referenceId = $request->query('reference_id')
            ?? $request->query('merchantTransId')
            ?? $request->query('orderId');

        $tx = $referenceId
            ? PaymentTransaction::where('reference_id', $referenceId)->latest()->first()
            : null;

        return view('payment.return', ['tx' => $tx]);
    }
}
```

### 9.5 `resources/views/payment/return.blade.php`

```blade
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title>Status Pembayaran</title>
    @if($tx && $tx->status === 'PENDING')
        <meta http-equiv="refresh" content="5">
    @endif
    <style>
        body { font-family: -apple-system, sans-serif; text-align: center; padding: 40px 20px; color: #333; }
        .card { max-width: 500px; margin: 0 auto; padding: 30px; border: 1px solid #e5e7eb; border-radius: 12px; }
        .status { font-size: 22px; font-weight: 600; margin: 20px 0; }
        .paid { color: #16a34a; } .pending { color: #f59e0b; } .failed { color: #dc2626; }
    </style>
</head>
<body>
<div class="card">
    @if($tx)
        @php
            $class = match($tx->status) { 'PAID' => 'paid', 'PENDING' => 'pending', default => 'failed' };
            $label = match($tx->status) {
                'PAID'    => '✅ Pembayaran Berhasil',
                'PENDING' => '⏳ Menunggu Konfirmasi',
                'EXPIRED' => '⏱️ Pembayaran Kedaluwarsa',
                default   => '❌ Pembayaran Gagal',
            };
        @endphp
        <div class="status {{ $class }}">{{ $label }}</div>
        <p>Kode: <strong>{{ $tx->reference_id }}</strong></p>
        <p>Jumlah: <strong>Rp {{ number_format((float) $tx->amount, 0, ',', '.') }}</strong></p>
        @if($tx->status === 'PAID')
            <p>E-ticket akan dikirim ke WhatsApp dan email Anda dalam beberapa detik.</p>
        @elseif($tx->status === 'PENDING')
            <p>Pembayaran sedang diproses. Halaman ini akan refresh otomatis.</p>
        @endif
    @else
        <div class="status pending">⏳ Menunggu Pembayaran</div>
        <p>Transaksi tidak ditemukan. Silakan kembali ke aplikasi.</p>
    @endif
</div>
</body>
</html>
```

---

## 10. 🚦 Routing & Keamanan

### 10.1 `routes/api.php`

```php
<?php

use App\Http\Controllers\Api\V1\AuthController;
use App\Http\Controllers\Api\V1\PaymentController;
use App\Http\Controllers\Api\WebhookController;
use Illuminate\Support\Facades\Route;

Route::prefix('v1')->group(function () {

    Route::post('/auth/login', [AuthController::class, 'login'])
        ->middleware('throttle:10,1');

    Route::middleware(['auth:sanctum', 'abilities:payment', 'throttle:payment-api'])->group(function () {
        Route::get('/auth/me',       [AuthController::class, 'me']);
        Route::post('/auth/refresh', [AuthController::class, 'refresh']);
        Route::post('/auth/logout',  [AuthController::class, 'logout']);

        Route::post('/payment/create',              [PaymentController::class, 'create']);
        Route::get('/payment/status/{referenceId}', [PaymentController::class, 'status']);
    });
});

// Webhook DANA (publik, diverifikasi signature RSA)
Route::post('/webhook/dana', [WebhookController::class, 'dana']);
```

### 10.2 `routes/web.php`

```php
use App\Http\Controllers\PaymentReturnController;
use Illuminate\Support\Facades\Route;

Route::get('/payment/return', PaymentReturnController::class)->name('payment.return');
```

### 10.3 CSRF & Middleware Sanctum

Rute `api/*` berada di group `api` sehingga tidak terkena CSRF. Bila Anda memindahkannya ke `web.php`, kecualikan:

```php
// Laravel 11+ — bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: ['api/*']);
})
```

```php
// Laravel 10 — app/Http/Middleware/VerifyCsrfToken.php
protected $except = ['api/*'];
```

Middleware `abilities:payment` adalah alias bawaan Sanctum. Jika tidak dikenali di Laravel 11+, daftarkan di `bootstrap/app.php`:

```php
$middleware->alias([
    'abilities' => \Laravel\Sanctum\Http\Middleware\CheckAbilities::class,
    'ability'   => \Laravel\Sanctum\Http\Middleware\CheckForAnyAbility::class,
]);
```

### 10.4 Verifikasi Rute

```bash
php artisan route:list | grep -E "auth|payment|webhook"
```

```text
POST      api/v1/auth/login
GET|HEAD  api/v1/auth/me             auth:sanctum
POST      api/v1/auth/refresh        auth:sanctum
POST      api/v1/auth/logout         auth:sanctum
POST      api/v1/payment/create      auth:sanctum
GET|HEAD  api/v1/payment/status/{referenceId}
POST      api/webhook/dana
GET|HEAD  payment/return
```

### 10.5 Hardening

| Kontrol | Implementasi |
|---|---|
| Rate limit per client | `throttle:payment-api` (§4.5) |
| Scope token | `abilities:payment` |
| IP allowlist | Batasi `/api/v1/*` ke IP server Ovisito (firewall/Nginx) |
| Webhook DANA | `PAYMENT_REQUIRE_SIGNATURE=true` + allowlist IP DANA bila tersedia |
| Isolasi data antar-client | Saat ada >1 client, batasi `status()` berdasarkan `metadata.client_id` |
| Log | Jangan mencatat secret/private key; `storage/logs` tidak world-readable |
| Transport | HTTPS wajib di production, termasuk callback ke Ovisito |

---

## 11. 🟦 Sisi `ovisitomcp.test`

### 11.1 `app/Services/Payment/PaymentClientService.php`

```php
<?php

namespace App\Services\Payment;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Http;

class PaymentClientService
{
    private string $baseUrl;
    private string $clientId;
    private string $clientSecret;

    public function __construct()
    {
        $this->baseUrl      = (string) config('services.payment.base_url');
        $this->clientId     = (string) config('services.payment.client_id');
        $this->clientSecret = (string) config('services.payment.client_secret');
    }

    public function createPayment(array $payload): array
    {
        $resp = Http::withToken($this->getToken())
            ->timeout(15)
            ->post("{$this->baseUrl}/payment/create", $payload);

        // Token kedaluwarsa/dicabut → login ulang sekali
        if ($resp->status() === 401) {
            Cache::forget('payment_api_token');
            $resp = Http::withToken($this->getToken())
                ->timeout(15)
                ->post("{$this->baseUrl}/payment/create", $payload);
        }

        if (!$resp->successful()) {
            throw new \RuntimeException('Payment create failed: ' . $resp->body());
        }

        return $resp->json('data') ?? [];
    }

    public function getStatus(string $referenceId): array
    {
        $resp = Http::withToken($this->getToken())
            ->timeout(10)
            ->get("{$this->baseUrl}/payment/status/{$referenceId}");

        return $resp->json('data') ?? [];
    }

    private function getToken(): string
    {
        return Cache::remember('payment_api_token', now()->addHours(6), function () {
            $resp = Http::timeout(10)->post("{$this->baseUrl}/auth/login", [
                'client_id'     => $this->clientId,
                'client_secret' => $this->clientSecret,
                'device_name'   => 'ovisito-core-server',
            ]);

            if (!$resp->successful()) {
                throw new \RuntimeException('Payment login failed: ' . $resp->body());
            }

            return $resp->json('data.access_token');
        });
    }
}
```

### 11.2 `app/Http/Middleware/VerifyPaymentWebhook.php`

Signature menutupi `timestamp.body`, dan timestamp **wajib** ada.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class VerifyPaymentWebhook
{
    public function handle(Request $request, Closure $next)
    {
        $signature = (string) $request->header('X-Payment-Signature', '');
        $timestamp = (int) $request->header('X-Payment-Timestamp', 0);
        $secret    = (string) config('services.payment.webhook_secret');

        if ($signature === '' || $timestamp === 0 || $secret === '') {
            return response()->json(['error' => 'Missing signature'], 401);
        }

        if (abs(time() - $timestamp) > 300) {
            return response()->json(['error' => 'Timestamp too old'], 401);
        }

        $expected = 'sha256=' . hash_hmac('sha256', $timestamp . '.' . $request->getContent(), $secret);

        if (!hash_equals($expected, $signature)) {
            return response()->json(['error' => 'Invalid signature'], 401);
        }

        return $next($request);
    }
}
```

### 11.3 `InternalPaymentCallbackController`

> [!NOTE]
> Controller ini memakai model `WebhookLog` (tabel dengan kolom `event_id`, `type`, `payload`, `processed_at`) dari pola webhook Ovisito yang sudah ada. Beri **unique index** pada `event_id` agar dua callback bersamaan tidak lolos keduanya.

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\transport_aceh\TransportBooking;
use App\Models\WebhookLog;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class InternalPaymentCallbackController extends Controller
{
    public function __invoke(Request $request)
    {
        $eventId = (string) $request->input('id');
        $type    = (string) $request->input('type');
        $data    = (array)  $request->input('data', []);

        if ($eventId === '' || empty($data['reference_id'])) {
            return response()->json(['error' => 'Invalid payload'], 422);
        }

        // Idempotency
        if (WebhookLog::where('event_id', $eventId)->exists()) {
            return response()->json(['received' => true, 'duplicate' => true]);
        }

        match ($type) {
            'payment.paid'    => $this->handlePaid($data),
            'payment.expired' => $this->handleExpired($data),
            default           => null,     // payment.failed: biarkan user mencoba lagi
        };

        WebhookLog::create([
            'event_id'     => $eventId,
            'type'         => $type,
            'payload'      => $request->all(),
            'processed_at' => now(),
        ]);

        return response()->json(['received' => true]);
    }

    protected function handlePaid(array $data): void
    {
        DB::transaction(function () use ($data) {
            $booking = TransportBooking::on('transports')
                ->where('booking_code', $data['reference_id'])
                ->lockForUpdate()
                ->first();

            if (!$booking || $booking->status === TransportBooking::STATUS_PAID) {
                return;
            }

            // Disarankan: pastikan nominal cocok sebelum menandai PAID
            // abort_if((float) $data['amount'] !== (float) $booking->total_price, 422);

            // ⚡ Eloquent update → Observer fire → WhatsApp terkirim
            $booking->update([
                'status'                 => TransportBooking::STATUS_PAID,
                'payment_provider'       => $data['provider'] ?? 'dana',
                'payment_transaction_id' => $data['external_id'] ?? null,
            ]);

            DB::connection('payments')->table('central_bookings')
                ->where('booking_code', $data['reference_id'])
                ->where('service_type', 'transport')
                ->update([
                    'payment_status' => 'paid',
                    'status'         => 'confirmed',
                    'paid_at'        => now(),
                ]);
        });
    }

    protected function handleExpired(array $data): void
    {
        TransportBooking::on('transports')
            ->where('booking_code', $data['reference_id'])
            ->where('status', 'pending')
            ->update(['status' => 'cancelled']);
    }
}
```

### 11.4 Route

```php
// ovisitomcp.test/routes/api.php
Route::middleware(\App\Http\Middleware\VerifyPaymentWebhook::class)
    ->post('/internal/payment-callback',
        \App\Http\Controllers\Api\InternalPaymentCallbackController::class);
```

### 11.5 `TransportWebController::payWithDana()`

```php
use App\Services\Payment\PaymentClientService;

public function payWithDana(string $code, PaymentClientService $payment)
{
    $booking = TransportBooking::on('transports')
        ->where('booking_code', $code)
        ->firstOrFail();
    abort_unless($booking->status === 'pending', 422);

    $notes = json_decode($booking->notes ?? '{}', true);

    $resp = $payment->createPayment([
        'reference_id' => $booking->booking_code,
        'amount'       => (float) $booking->total_price,
        'provider'     => 'dana',
        'return_url'   => route('payment.return', ['code' => $booking->booking_code]),
        'customer'     => [
            'name'  => $notes['passenger_name']  ?? null,
            'phone' => $notes['passenger_phone'] ?? null,
            'email' => $notes['passenger_email'] ?? null,
        ],
    ]);

    if (empty($resp['checkout_url'])) {
        return back()->withErrors(['payment' => 'Gagal membuat pembayaran.']);
    }

    // Status masih pending → Observer belum terpicu
    $booking->update([
        'payment_provider'       => 'dana',
        'payment_transaction_id' => $resp['external_id'],
        'payment_url'            => $resp['checkout_url'],
    ]);

    return redirect()->away($resp['checkout_url']);
}
```

---

## 12. ⏱️ Scheduler & Reliability

Tiga lapis pengaman memastikan tidak ada pembayaran yang "menggantung":

| Command | Frekuensi | Fungsi |
|---|---|---|
| `payment:retry-callbacks` | tiap menit | Mengirim ulang callback ke Ovisito yang gagal (exponential backoff, maks. 10×) |
| `payment:poll-pending` | tiap 2 menit | Menanyakan status ke DANA bila webhook hilang |
| `payment:expire-pending` | tiap 5 menit | Menandai transaksi lewat `expires_at` sebagai `EXPIRED` |

### 12.1 `RetryFailedCallbacks`

```php
<?php

namespace App\Console\Commands;

use App\Models\PaymentTransaction;
use App\Services\OvisitoCallbackService;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class RetryFailedCallbacks extends Command
{
    protected $signature   = 'payment:retry-callbacks';
    protected $description = 'Kirim ulang callback ke Ovisito yang sebelumnya gagal';

    public function handle(OvisitoCallbackService $callback): int
    {
        $rows = DB::table('failed_callbacks')
            ->whereNull('resolved_at')
            ->where('attempts', '<', 10)
            ->where('next_attempt_at', '<=', now())
            ->limit(50)
            ->get();

        foreach ($rows as $row) {
            $tx = PaymentTransaction::where('uuid', $row->transaction_uuid)->first();

            if (!$tx) {
                DB::table('failed_callbacks')->where('id', $row->id)->update([
                    'resolved_at' => now(),
                    'last_error'  => 'Transaction not found',
                    'updated_at'  => now(),
                ]);
                continue;
            }

            $status = json_decode($row->payload, true)['data']['status'] ?? strtolower($tx->status);

            // recordFailure=false: attempts dikelola di sini, bukan baris baru
            if ($callback->sendResult($tx, $status, false)) {
                DB::table('failed_callbacks')->where('id', $row->id)->update([
                    'resolved_at' => now(),
                    'updated_at'  => now(),
                ]);
                $this->info("✓ {$tx->reference_id}");
            } else {
                $attempts = $row->attempts + 1;
                DB::table('failed_callbacks')->where('id', $row->id)->update([
                    'attempts'        => $attempts,
                    'next_attempt_at' => now()->addMinutes(min(60, 2 ** $attempts)),
                    'updated_at'      => now(),
                ]);
                $this->warn("✗ {$tx->reference_id} (percobaan {$attempts})");
            }
        }

        return self::SUCCESS;
    }
}
```

### 12.2 `ExpirePendingTransactions`

```php
<?php

namespace App\Console\Commands;

use App\Models\PaymentTransaction;
use App\Services\OvisitoCallbackService;
use Illuminate\Console\Command;

class ExpirePendingTransactions extends Command
{
    protected $signature   = 'payment:expire-pending';
    protected $description = 'Tandai transaksi PENDING yang melewati expires_at';

    public function handle(OvisitoCallbackService $callback): int
    {
        $expired = PaymentTransaction::where('status', 'PENDING')
            ->where('expires_at', '<', now())
            ->limit(100)
            ->get();

        foreach ($expired as $tx) {
            $tx->update(['status' => 'EXPIRED']);
            $callback->sendResult($tx, 'expired');
            $this->line("✓ {$tx->reference_id}");
        }

        return self::SUCCESS;
    }
}
```

### 12.3 `PollPendingTransactions`

```php
<?php

namespace App\Console\Commands;

use App\Models\PaymentTransaction;
use App\Services\OvisitoCallbackService;
use App\Services\PaymentManager;
use Illuminate\Console\Command;

class PollPendingTransactions extends Command
{
    protected $signature   = 'payment:poll-pending';
    protected $description = 'Query DANA untuk transaksi PENDING yang webhook-nya belum masuk';

    public function handle(PaymentManager $manager, OvisitoCallbackService $callback): int
    {
        $pending = PaymentTransaction::where('status', 'PENDING')
            ->where('created_at', '<', now()->subMinutes(2))
            ->where('expires_at', '>', now())
            ->where('provider', 'dana')
            ->limit(20)
            ->get();

        foreach ($pending as $tx) {
            try {
                $result = $manager->driver('dana')->queryTransaction($tx->external_id);

                if ($result['status'] === 'paid') {
                    $tx->update(['status' => 'PAID', 'paid_at' => now()]);
                    $callback->sendResult($tx, 'paid');
                    $this->info("✓ {$tx->reference_id} → PAID (via query)");
                } elseif (in_array($result['status'], ['expired', 'failed'], true)) {
                    $tx->update(['status' => strtoupper($result['status'])]);
                    $callback->sendResult($tx, $result['status']);
                }
            } catch (\Throwable $e) {
                $this->error("✗ {$tx->reference_id}: {$e->getMessage()}");
            }
        }

        return self::SUCCESS;
    }
}
```

### 12.4 Penjadwalan

```php
// Laravel 11+ — routes/console.php
use Illuminate\Support\Facades\Schedule;

Schedule::command('payment:retry-callbacks')->everyMinute()->withoutOverlapping();
Schedule::command('payment:expire-pending')->everyFiveMinutes()->withoutOverlapping();
Schedule::command('payment:poll-pending')->everyTwoMinutes()->withoutOverlapping();
```

Laravel 10: letakkan tiga baris yang sama di `app/Console/Kernel.php` → `schedule()` dengan `$schedule->command(...)`.

```bash
# Cron server
* * * * * cd /var/www/payment && php artisan schedule:run >> /dev/null 2>&1
```

---

## 13. 🧪 Pengujian End-to-End

> [!TIP]
> Mode strict (`PAYMENT_REQUIRE_SIGNATURE=true`) menolak webhook tanpa signature valid. Untuk uji `curl` manual di lokal, set sementara ke `false`, lalu **kembalikan ke `true` sebelum deploy**.

### 13.1 Ringkasan Skenario

| # | Skenario | Hasil yang diharapkan |
|---|---|---|
| 1 | Endpoint webhook hidup | HTTP 200, `resultStatus: "S"` (non-strict) |
| 2 | Login ClientApp | `access_token` diterbitkan; secret salah → 401 `INVALID_CREDENTIALS` |
| 3 | Create payment (Bearer) | `checkout_url` DANA; tanpa token → 401 |
| 4 | Bayar di cashier sandbox | Webhook masuk, status `PAID` |
| 5 | Webhook + callback | `ovisito_synced_at` terisi, WA terkirim |
| 6 | Idempotency (webhook 3×) | Callback & WA hanya 1× |
| 7 | Callback gagal → retry | Satu baris `failed_callbacks` → `resolved_at` terisi |
| 8 | Expire | `EXPIRED`, booking `cancelled` |
| 9 | Signature palsu | HTTP 401, `resultStatus: "F"` |

### 13.2 Test 1 — Webhook Hidup

```bash
curl -X POST http://payment.test/api/webhook/dana \
  -H "Content-Type: application/json" \
  -d '{"response":{"body":{"acquirementId":"HEALTH","latestTransactionStatus":"SUCCESS"}}}'
```

### 13.3 Test 2 — Login

```bash
curl -X POST http://payment.test/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"client_id":"ovisito-core","client_secret":"<secret>"}'
```

```json
{"success":true,"data":{"access_token":"1|...","token_type":"Bearer","expires_at":"..."}}
```

Kasus error (secret salah) → `401 INVALID_CREDENTIALS`.

### 13.4 Test 3 — Create Payment

```bash
TOKEN=$(curl -s -X POST http://payment.test/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"client_id":"ovisito-core","client_secret":"<secret>"}' | jq -r '.data.access_token')

curl -X POST http://payment.test/api/v1/payment/create \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "reference_id": "TRX-TEST-001",
    "amount": 150000,
    "provider": "dana",
    "return_url": "http://payment.test/payment/return"
  }'
```

### 13.5 Test 4–5 — Bayar & Verifikasi

1. Buka `checkout_url` → bayar di cashier DANA sandbox.
2. Pantau log:

```bash
tail -f storage/logs/laravel.log | grep -iE "webhook|callback"
```

3. Cek database:

```sql
-- payment.test
SELECT reference_id, status, paid_at, ovisito_synced_at
FROM payment_transactions WHERE reference_id = 'TRX-TEST-001';
-- Harapan: PAID, paid_at & ovisito_synced_at terisi

-- ovisitomcp.test
SELECT booking_code, status, payment_provider, wa_sent_at
FROM transport_bookings WHERE booking_code = 'TRX-TEST-001';
-- Harapan: paid, dana, wa_sent_at tidak null
```

### 13.6 Test 6 — Idempotency

```bash
ACQ="<acquirement_id dari respons create>"
for i in 1 2 3; do
  curl -X POST http://payment.test/api/webhook/dana \
    -H "Content-Type: application/json" \
    -d "{\"response\":{\"body\":{\"acquirementId\":\"$ACQ\",\"latestTransactionStatus\":\"SUCCESS\"}}}"
done
```

Harapan: log `[Callback] OK` hanya 1×, `ovisito_synced_at` tidak berubah, WA terkirim 1×.

### 13.7 Test 7 — Callback Gagal → Retry

1. Matikan server `ovisitomcp.test`.
2. Picu pembayaran → cek `failed_callbacks` (1 baris).
3. Nyalakan kembali server, jalankan `php artisan payment:retry-callbacks`.
4. Harapan: `✓ TRX-TEST-001` dan `resolved_at` terisi.

### 13.8 Test 8 — Expire

Buat transaksi, jangan dibayar, tunggu lewat `expires_at` (default 60 menit), lalu:

```bash
php artisan payment:expire-pending
```

Harapan: status `EXPIRED`, callback `payment.expired` terkirim, booking `cancelled`.

### 13.9 Test 9 — Signature Palsu

```bash
curl -X POST http://payment.test/api/webhook/dana \
  -H "X-SIGNATURE: invalid_signature" \
  -H "Content-Type: application/json" \
  -d '{"response":{"body":{"acquirementId":"TEST"}}}'
```

Harapan: HTTP 401, `resultStatus: "F"`.

---

## 14. ✅ Deployment Checklist

### 14.1 Sebelum Deploy

| # | Item | ☐ |
|---|---|---|
| 1 | `.env` `payment.test` lengkap (DANA + ClientApp + Callback) | ☐ |
| 2 | `.env` `ovisitomcp.test` lengkap (`PAYMENT_*`) | ☐ |
| 3 | `DANA_ORDER_NOTIFY_URL` & `DANA_PAY_RETURN_URL` = HTTPS publik | ☐ |
| 4 | `DANA_PUB_KEY` valid (tidak kosong) | ☐ |
| 5 | `PAYMENT_REQUIRE_SIGNATURE=true` | ☐ |
| 6 | `OVISITO_CLIENT_SECRET` identik di kedua project | ☐ |
| 7 | `OVISITO_WEBHOOK_SECRET` identik di kedua project | ☐ |
| 8 | `php artisan migrate --force` sudah dijalankan | ☐ |
| 9 | `db:seed --class=ClientAppSeeder` sudah dijalankan | ☐ |
| 10 | Tabel `personal_access_tokens` ada | ☐ |
| 11 | Unique index pada `webhook_logs.event_id` (Ovisito) | ☐ |
| 12 | `route:list` menampilkan rute auth/payment/webhook | ☐ |
| 13 | Cron `schedule:run` aktif | ☐ |
| 14 | `APP_ENV=production`, `APP_DEBUG=false` | ☐ |
| 15 | Kredensial DANA production (bukan sandbox) sudah dipasang | ☐ |
| 16 | `storage/logs` tidak world-readable | ☐ |

### 14.2 Setelah Deploy

```bash
php artisan config:cache
php artisan route:cache

# Login
curl -X POST https://payment.ovisito.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"client_id":"ovisito-core","client_secret":"<production-secret>"}'

# Lalu: create payment bernilai kecil → bayar → verifikasi webhook → callback → WA
```

---

## 15. 🔁 Menambah Gateway Baru

Contoh: Flip, GoPay, BSI VA.

**1. Buat class gateway** — `app/Services/Gateways/FlipGateway.php`

```php
class FlipGateway implements PaymentGatewayInterface
{
    public function getName(): string { return 'flip'; }

    public function createTransaction(array $data): array
    {
        // Panggil API Flip → ['external_id', 'checkout_url', 'expires_at']
    }

    public function handleWebhook(Request $request): array
    {
        // Verifikasi signature Flip → ['status', 'reference_id', 'external_id', 'amount', 'raw']
    }
}
```

**2. Daftarkan di `PaymentManager`**

```php
'flip'  => new FlipGateway(),
'gopay' => new GopayGateway(),
```

**3. Tambah route webhook**

```php
Route::post('/webhook/flip',  [WebhookController::class, 'flip']);
Route::post('/webhook/gopay', [WebhookController::class, 'gopay']);
```

**4. Ekstrak logika generik** — refactor `WebhookController::dana()` menjadi `handleGenericWebhook(Request $request, string $provider)`; method `dana`, `flip`, `gopay` cukup mendelegasikannya.

> Tidak ada perubahan pada `AuthController`, `PaymentController`, `OvisitoCallbackService`, maupun skema database.

---

## 16. 🩺 Troubleshooting

| Gejala | Kemungkinan penyebab | Tindakan |
|---|---|---|
| Login 401 `INVALID_CREDENTIALS` | Seeder belum dijalankan / secret berbeda dengan `.env` | Jalankan `ClientAppSeeder`, samakan secret |
| Login 422 | Payload tidak valid | Pastikan `client_id` & `client_secret` terkirim |
| Create 401 | Token kedaluwarsa/dicabut | Hapus cache `payment_api_token`, login ulang |
| Create 403 | Token tanpa ability `payment` | Terbitkan ulang token lewat `/auth/login` |
| Create 502 `GATEWAY_ERROR` | Kredensial/private key DANA salah, format `\n` pada key | Periksa `.env`; lihat log `[DANA] createOrder` |
| Webhook tidak masuk | URL notify bukan HTTPS publik / tunnel mati | Cek ngrok & `DANA_ORDER_NOTIFY_URL` |
| Webhook 401 | Signature gagal / `DANA_PUB_KEY` kosong (strict) | Isi public key DANA; kirim contoh `X-SIGNATURE` untuk dianalisis |
| Callback ke Ovisito 401 | Webhook secret beda / jam server selisih >5 menit | Samakan secret, sinkronkan NTP |
| `failed_callbacks` menumpuk | Ovisito down / validasi gagal | Cek log Ovisito, jalankan `payment:retry-callbacks` |
| Nominal salah di cashier | Satuan `orderAmount.value` | Lihat catatan di bawah |

> [!CAUTION]
> **Verifikasi terhadap dokumentasi resmi DANA sebelum production:**
> 1. **Satuan nominal** — sebagian API DANA memakai satuan sen (Rp 1.000 → `100000`). Uji dengan transaksi kecil di sandbox, lalu sesuaikan konversi di `DanaGateway::createTransaction()` dan pembacaan `orderAmount.value` pada webhook.
> 2. **Format signature webhook** — dokumen ini memverifikasi RSA-SHA256 atas *raw body*. Cocokkan dengan *string-to-sign* yang ditetapkan DANA untuk versi API yang dipakai.
> 3. **Nama method package** — pastikan `DANAPay::createOrder()` dan `queryOrder()` sesuai versi `otnansirk/laravel-dana` yang terpasang.

---

## 17. 📈 Perubahan v1 → v2

### 17.1 Perubahan Arsitektur

| Area | v1 (API Key) | v2 (Sanctum + HMAC) |
|---|---|---|
| Auth Ovisito → payment | `X-API-KEY` statis | Sanctum Bearer (login / refresh / logout) |
| Masa berlaku token | Tak terbatas | 30 hari, dapat di-refresh & dicabut |
| Multi-client | ❌ Terbatas | ✅ Tabel `client_apps` + rate limit per client |
| Callback payment → Ovisito | `X-Signature` custom | `X-Payment-Signature` (selaras `X-Ovisito-Signature`) |
| Format callback | Flat | Envelope `{id, type, created_at, data}` |
| Webhook DANA | Signature DANA | Signature DANA (tidak berubah) |
| Konsistensi dengan `ovisitomcp.test` | 🟡 Berbeda | ✅ Pola identik |

### 17.2 Perbaikan Teknis pada Edisi Ini

| # | Perbaikan | Alasan |
|---|---|---|
| 1 | Mode `PAYMENT_REQUIRE_SIGNATURE` (strict default) | Draf lama menerima webhook tanpa signature → celah pemalsuan status bayar |
| 2 | Signature callback mencakup timestamp (`timestamp.body`); timestamp wajib | Sebelumnya timestamp tidak ikut ditandatangani dan boleh kosong → rentan replay |
| 3 | ID event deterministik `evt_{uuid}_{status}` | Sebelumnya setiap retry membuat ID baru sehingga idempotency receiver tidak efektif |
| 4 | `$recordFailure` pada `sendResult()` | Command retry menggandakan baris `failed_callbacks` |
| 5 | `device_name` opsional pada login | Contoh `curl` tidak mengirimnya → 422 |
| 6 | `abilities:payment` + rate limiter per client | Scope token & pemanfaatan kolom `rate_limit_per_minute` |
| 7 | `needsCallback()` mencakup `EXPIRED`/`FAILED` | Callback non-paid ikut dijamin tersinkron |
| 8 | Unique index `(provider, external_id)` | Mencegah duplikasi transaksi gateway |
| 9 | `reference_id` disisipkan ke return URL | DANA tidak selalu mengirim parameter yang dapat dipakai halaman return |
| 10 | `<meta refresh>` dipindah ke `<head>` | HTML valid, perilaku konsisten |
| 11 | Pesan error gateway disamarkan saat `APP_DEBUG=false` | Mencegah kebocoran detail internal |
| 12 | Batas bawah 24 jam pada expire dihapus | Transaksi lama yang terlewat tetap ditutup |
| 13 | Kredensial contoh diganti placeholder | Menghindari nilai yang menyerupai secret asli |

---

## 18. 🗓️ Rencana Eksekusi

| Fase | Cakupan | Estimasi | Kriteria selesai |
|---|---|---|---|
| 1 | §3–§5 struktur, migration, DB | 15 mnt | `migrate` & seeder sukses |
| 2 | §6–§9 model, service, controller | 45 mnt | Kode terpasang tanpa error |
| 3 | §10 routing | 5 mnt | `route:list` sesuai |
| 4 | §11 sisi `ovisitomcp.test` | 30 mnt | `PaymentClientService` & receiver siap |
| 5 | Test 1–3 | 15 mnt | `checkout_url` DANA muncul |
| 6 | Ngrok + Test 4–5 | 20 mnt | Bayar sandbox → `PAID` → WA terkirim |
| 7 | Test 6–9 | 20 mnt | Idempotency, retry, expire, signature lolos |
| 8 | §12 scheduler + §14 checklist | 15 mnt | Cron aktif, checklist lengkap |
| | **Total** | **± 2 jam 45 mnt** | |

---

<div align="center">

**Dokumen ini hidup** — perbarui versi dan tabel perubahan setiap ada modifikasi arsitektur atau penambahan gateway.

Bila ada error saat eksekusi (mis. `DANAPay::createOrder()` tidak cocok dengan versi SDK, atau signature webhook gagal), kirim stack trace / contoh header `X-SIGNATURE` untuk iterasi berikutnya.

*© 2026 PT Balai Digital Teknologi · Ovisito*

</div>
