# Payment Service Terpusat — `payment.test`

**Panduan Implementasi Integrasi DANA untuk Ekosistem Ovisito**

| | |
|---|---|
| **Dokumen** | Technical Implementation Guide |
| **Versi** | 1.1.0 |
| **Tanggal** | 5 November 2026 |
| **Pemilik** | PT Balai Digital Teknologi — Tim Engineering Ovisito |
| **Stack** | PHP 8.2+, Laravel 10/11, MySQL 8+, `otnansirk/laravel-dana` |
| **Status** | Siap dieksekusi (sandbox → production) |

---

## Daftar Isi

1. [Ringkasan Eksekutif](#1-ringkasan-eksekutif)
2. [Arsitektur](#2-arsitektur)
3. [Prasyarat](#3-prasyarat)
4. [Struktur Folder](#4-struktur-folder)
5. [Konfigurasi Environment](#5-konfigurasi-environment)
6. [Database](#6-database)
7. [Model Eloquent](#7-model-eloquent)
8. [Kontrak Gateway](#8-kontrak-gateway)
9. [Implementasi Gateway DANA](#9-implementasi-gateway-dana)
10. [Service Layer](#10-service-layer)
11. [Controller & Middleware](#11-controller--middleware)
12. [Routing & Keamanan](#12-routing--keamanan)
13. [Receiver Callback di Ovisito](#13-receiver-callback-di-ovisito)
14. [Scheduler: Retry, Expire, Polling](#14-scheduler-retry-expire-polling)
15. [Panduan Pengujian](#15-panduan-pengujian)
16. [Deployment Checklist](#16-deployment-checklist)
17. [Menambah Gateway Baru](#17-menambah-gateway-baru)
18. [Troubleshooting](#18-troubleshooting)
19. [Catatan Perubahan dari Draf Awal](#19-catatan-perubahan-dari-draf-awal)
20. [Rencana Eksekusi](#20-rencana-eksekusi)

---

## 1. Ringkasan Eksekutif

`payment.test` adalah **Payment Service Terpusat** yang menangani pembayaran DANA (dan gateway lain di masa depan) atas nama `ovisito.com`.

**Alur target:**

1. User memesan tiket di `ovisito.com/transport` → sistem menerbitkan kode booking.
2. User memilih **Bayar DANA** → diarahkan ke `payment.test` → DANA cashier.
3. DANA mengirim webhook ke `payment.test` → `payment.test` mengirim callback ter-signature ke `ovisito.com` → notifikasi WhatsApp terkirim otomatis.

**Prinsip desain:**

| Prinsip | Penjelasan |
|---|---|
| Separation of concerns | `payment.test` hanya tahu: terima request, buat transaksi, verifikasi webhook, kirim callback. Tidak tahu soal booking, observer, atau WhatsApp. |
| Business logic di Ovisito | `ovisito.com` tetap memegang status booking dan notifikasi. |
| Idempotent | Webhook/callback yang dikirim berulang tidak menghasilkan efek ganda. |
| Reliable | Callback gagal di-retry; webhook hilang ditutup polling; transaksi kedaluwarsa ditandai otomatis. |
| Extensible | Gateway baru cukup menambah satu class tanpa mengubah core. |

---

## 2. Arsitektur

```text
┌──────────────────────────────────────────────────────────────────┐
│  ovisito.com / ovisitomcp.test (Laravel — EXISTING)              │
│  • Booking tiket, kode booking, DB aceh_transports               │
│  • TransportBookingObserver → WhatsAppService (Fonnte)           │
│  • Endpoint internal: POST /api/internal/payment-callback        │
└────────────────────────┬─────────────────────────────────────────┘
                         │ 1) POST /api/payment/create  (X-API-KEY)
                         │ 2) ◄── { checkout_url, transaction_id }
                         │
                         │ 5) ◄── callback (HMAC) saat PAID/EXPIRED/FAILED
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  payment.ovisito.com = payment.test (Laravel — BARU)             │
│  • Package: otnansirk/laravel-dana                               │
│  • DB: payment_gateway (payment_transactions, failed_callbacks)  │
│  • Endpoint:                                                     │
│    ├─ POST /api/payment/create        (dari Ovisito)             │
│    ├─ GET  /api/payment/status/{ref}  (dari Ovisito)             │
│    ├─ POST /api/webhook/dana          (dari DANA)                │
│    └─ GET  /payment/return            (dari browser user)        │
└────────────────────────┬─────────────────────────────────────────┘
                         │ 3) createOrder → acquirementId + checkoutUrl
                         │ 4) ◄── webhook Finish Notify
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│  DANA Sandbox / Production                                       │
└──────────────────────────────────────────────────────────────────┘
```

**Status transaksi:** `PENDING` → `PAID` | `EXPIRED` | `FAILED` | `CANCELLED`

**Dua mekanisme autentikasi antar-service:**

| Arah | Mekanisme | Tujuan |
|---|---|---|
| Ovisito → payment.test | Header `X-API-KEY` | Membatasi siapa yang boleh membuat transaksi |
| payment.test → Ovisito | HMAC-SHA256 + timestamp + nonce | Menjamin keaslian callback dan mencegah replay |
| DANA → payment.test | Signature RSA (`X-SIGNATURE`) | Menjamin webhook benar-benar dari DANA |

---

## 3. Prasyarat

### 3.1 Perangkat

| Item | Cek dengan |
|---|---|
| PHP 8.2+ | `php -v` |
| Composer | `composer -V` |
| MySQL 8+ | `mysql --version` |
| Laravel 10/11 di `payment.test` | `php artisan --version` |
| Package `otnansirk/laravel-dana` | `composer show otnansirk/laravel-dana` |
| File `config/dana.php` | `ls config/dana.php` |
| Tunnel HTTPS publik (lokal) | `ngrok http 8000` atau `cloudflared` |

### 3.2 Dari DANA Sandbox

1. Login ke `dashboard.dana.id/sandbox`.
2. Buat **Merchant**, catat `merchantId`, `clientId`, `clientSecret`.
3. Generate **RSA key pair**: simpan private key, unggah public key.
4. Unduh **DANA Public Key** (untuk verifikasi webhook).

### 3.3 Generate secret antar-service

```bash
# Jalankan dua kali: satu untuk PAYMENT_API_KEY, satu untuk HMAC_SECRET
php -r "echo bin2hex(random_bytes(32)) . PHP_EOL;"
```

> ⚠️ Simpan hanya di `.env`. Jangan commit ke repository, jangan tempel di chat/ticket.

---

## 4. Struktur Folder

```text
payment.test/
├── app/
│   ├── Console/Commands/
│   │   ├── RetryFailedCallbacks.php
│   │   ├── ExpirePendingTransactions.php
│   │   └── PollPendingTransactions.php
│   ├── Contracts/
│   │   └── PaymentGatewayInterface.php
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── Api/
│   │   │   │   ├── PaymentController.php
│   │   │   │   └── WebhookController.php
│   │   │   └── PaymentReturnController.php
│   │   └── Middleware/
│   │       └── VerifyPaymentApiKey.php
│   ├── Models/
│   │   └── PaymentTransaction.php
│   ├── Providers/
│   │   └── AppServiceProvider.php          ← binding HmacSigner
│   └── Services/
│       ├── Gateways/DanaGateway.php
│       ├── PaymentManager.php
│       ├── OvisitoCallbackService.php
│       └── HmacSigner.php
├── config/
│   ├── dana.php                            ← sudah ada
│   └── services.php                        ← tambah ovisito + payment
├── database/migrations/
│   └── 2026_11_05_000000_create_payment_transactions_table.php
├── resources/views/payment/return.blade.php
├── routes/{api.php, web.php, console.php}
└── .env
```

---

## 5. Konfigurasi Environment

### 5.1 `.env` di `payment.test`

```env
APP_NAME="Ovisito Payment"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://payment.test

# ─── Database (terpisah dari Ovisito) ─────────────────────────────
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=payment_gateway
DB_USERNAME=root
DB_PASSWORD=

# ─── DANA (sesuai config/dana.php) ────────────────────────────────
DANA_ENV=sandbox
DANA_ACTIVE=true
DANA_API_URL=https://api-sandbox.saas.dana.id
DANA_WEB_URL=https://m.sandbox.dana.id

# Penulisan MARCHANT (typo) disengaja — mengikuti package
DANA_MARCHANT_ID=<merchant-id-sandbox>
DANA_CLIENT_ID=<client-id-sandbox>
DANA_CLIENT_SECRET=<client-secret-sandbox>

# RSA keys — tanda kutip ganda + \n
DANA_PRIVATE_KEY="-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----"
DANA_PUB_KEY="-----BEGIN PUBLIC KEY-----\n...\n-----END PUBLIC KEY-----"

# Webhook & Return URL — WAJIB HTTPS publik (lokal: ngrok)
DANA_ORDER_NOTIFY_URL=https://<tunnel>.ngrok.io/api/webhook/dana
DANA_PAY_RETURN_URL=https://<tunnel>.ngrok.io/payment/return

# ─── Komunikasi ke Ovisito ────────────────────────────────────────
OVISITO_BASE_URL=https://ovisitomcp.test
PAYMENT_API_KEY=<hex-64-karakter>
HMAC_SECRET=<hex-64-karakter>          # HARUS SAMA dengan di Ovisito

# Production: true. Lokal/sandbox awal boleh false.
PAYMENT_REQUIRE_SIGNATURE=true
```

### 5.2 `config/services.php`

```php
'ovisito' => [
    'base_url'      => env('OVISITO_BASE_URL', 'http://ovisitomcp.test'),
    'callback_path' => '/api/internal/payment-callback',
    'api_key'       => env('PAYMENT_API_KEY'),
    'hmac_secret'   => env('HMAC_SECRET'),
],

'payment' => [
    'require_webhook_signature' => env('PAYMENT_REQUIRE_SIGNATURE', true),
],
```

### 5.3 Verifikasi `config/dana.php`

```php
"merchant_id"     => env("DANA_MARCHANT_ID", ""),     // typo disengaja
"ssh_public_key"  => env("DANA_PUB_KEY", ""),          // public key DANA
"ssh_private_key" => env("DANA_PRIVATE_KEY", ""),      // private key Anda
```

### 5.4 Binding `HmacSigner`

`OvisitoCallbackService` membutuhkan `HmacSigner` yang diberi secret. Daftarkan di `app/Providers/AppServiceProvider.php`:

```php
use App\Services\HmacSigner;

public function register(): void
{
    $this->app->singleton(HmacSigner::class, fn () => new HmacSigner(
        (string) config('services.ovisito.hmac_secret')
    ));
}
```

---

## 6. Database

### 6.1 Buat database

```bash
mysql -u root -e "CREATE DATABASE payment_gateway CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

### 6.2 Migration

`database/migrations/2026_11_05_000000_create_payment_transactions_table.php`

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

            // Identitas dari sistem inti
            $table->string('reference_id', 100)->index();   // booking_code
            $table->string('external_id', 100)->nullable(); // acquirementId DANA

            // Provider
            $table->string('provider', 30)->index();        // dana, flip, bsi, gopay
            $table->decimal('amount', 15, 2);
            $table->char('currency', 3)->default('IDR');
            $table->string('status', 20)->default('PENDING')->index();

            // Redirect
            $table->string('checkout_url', 500)->nullable();
            $table->timestamp('expires_at')->nullable();

            $table->json('metadata')->nullable();

            // Waktu
            $table->timestamp('paid_at')->nullable();
            $table->timestamp('ovisito_synced_at')->nullable()
                ->comment('null = callback ke Ovisito belum berhasil');
            $table->timestamps();

            $table->unique(['provider', 'external_id']);
            $table->index(['provider', 'status']);
            $table->index(['status', 'expires_at']);
            $table->index(['ovisito_synced_at', 'paid_at']);
        });

        Schema::create('failed_callbacks', function (Blueprint $table) {
            $table->id();
            $table->uuid('transaction_uuid')->index();
            $table->string('payload', 2000);
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

### 6.3 Jalankan & verifikasi

```bash
php artisan migrate
php artisan tinker
>>> Schema::hasTable('payment_transactions');   // true
>>> Schema::hasTable('failed_callbacks');       // true
```

---

## 7. Model Eloquent

`app/Models/PaymentTransaction.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Str;

class PaymentTransaction extends Model
{
    protected $table = 'payment_transactions';

    protected $fillable = [
        'uuid', 'reference_id', 'external_id', 'provider', 'amount',
        'currency', 'status', 'checkout_url', 'expires_at', 'metadata',
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
        static::creating(function (self $model) {
            if (empty($model->uuid)) {
                $model->uuid = (string) Str::uuid();
            }
        });
    }

    public function isPending(): bool { return $this->status === 'PENDING'; }
    public function isPaid(): bool    { return $this->status === 'PAID'; }

    public function needsCallback(): bool
    {
        return $this->ovisito_synced_at === null
            && in_array($this->status, ['PAID', 'EXPIRED', 'FAILED'], true);
    }
}
```

---

## 8. Kontrak Gateway

`app/Contracts/PaymentGatewayInterface.php`

```php
<?php

namespace App\Contracts;

use Illuminate\Http\Request;

interface PaymentGatewayInterface
{
    /** Nama provider, mis. 'dana'. */
    public function getName(): string;

    /**
     * Membuat transaksi di gateway.
     *
     * @param array $data {reference_id, amount, return_url, customer[]}
     * @return array {external_id, checkout_url, expires_at(Carbon)}
     */
    public function createTransaction(array $data): array;

    /**
     * Memproses webhook gateway.
     *
     * @return array {status: paid|expired|failed|unknown,
     *                reference_id, external_id, amount, raw}
     * @throws \Exception jika signature tidak valid
     */
    public function handleWebhook(Request $request): array;
}
```

---

## 9. Implementasi Gateway DANA

`app/Services/Gateways/DanaGateway.php`

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

    // ── 1) Create order ───────────────────────────────────────────
    public function createTransaction(array $data): array
    {
        $expiry    = now()->addMinutes($this->expiredAfter);
        $returnUrl = $data['return_url'] ?? config('dana.pay_return_url');
        // Sisipkan reference_id agar halaman return bisa menemukan transaksi
        $returnUrl .= (str_contains($returnUrl, '?') ? '&' : '?')
                    . 'reference_id=' . urlencode($data['reference_id']);

        $payload = [
            'order' => [
                'orderTitle'        => 'Pembayaran ' . $data['reference_id'],
                'orderAmount'       => [
                    'currency' => 'IDR',
                    'value'    => (string) $data['amount'],   // lihat catatan §18
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

    // ── 2) Handle webhook ─────────────────────────────────────────
    public function handleWebhook(Request $request): array
    {
        $signature = (string) $request->header('X-SIGNATURE', '');
        $rawBody   = $request->getContent();

        // Mode strict (default): signature WAJIB valid.
        // Mode non-strict (hanya sandbox awal): signature divalidasi bila ada.
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

    // ── 3) Query status (fallback webhook hilang) ─────────────────
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

    // ── 4) Verifikasi signature ───────────────────────────────────
    private function verifySignature(string $rawBody, string $signature): bool
    {
        if ($this->danaPublicKey === '') {
            if ($this->strictSignature) {
                Log::error('[DANA] DANA_PUB_KEY kosong pada mode strict — webhook ditolak');
                return false;
            }
            Log::warning('[DANA] DANA_PUB_KEY kosong — signature bypass (non-strict)');
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

## 10. Service Layer

### 10.1 `PaymentManager` (factory)

`app/Services/PaymentManager.php`

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
            // 'bsi'   => new BsiGateway(),
            // 'gopay' => new GopayGateway(),
            default => throw new InvalidArgumentException("Provider '{$provider}' tidak didukung."),
        };
    }

    public function supportsQuery(string $provider): bool
    {
        return in_array(strtolower($provider), ['dana'], true);
    }
}
```

### 10.2 `HmacSigner`

`app/Services/HmacSigner.php`

```php
<?php

namespace App\Services;

class HmacSigner
{
    public function __construct(private readonly string $secret) {}

    /**
     * HMAC-SHA256( METHOD \n PATH \n TIMESTAMP \n NONCE \n SHA256(BODY) )
     */
    public function sign(string $method, string $path, int $timestamp, string $nonce, string $body): string
    {
        $stringToSign = implode("\n", [
            strtoupper($method),
            $path,
            (string) $timestamp,
            $nonce,
            hash('sha256', $body),
        ]);

        return hash_hmac('sha256', $stringToSign, $this->secret);
    }
}
```

### 10.3 `OvisitoCallbackService`

`app/Services/OvisitoCallbackService.php`

Parameter `$recordFailure` mencegah duplikasi baris `failed_callbacks` ketika dipanggil dari command retry.

```php
<?php

namespace App\Services;

use App\Models\PaymentTransaction;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;

class OvisitoCallbackService
{
    public function __construct(private readonly HmacSigner $signer) {}

    /**
     * Kirim hasil pembayaran ke Ovisito.
     *
     * @param bool $recordFailure true = catat ke failed_callbacks bila gagal
     *                            (false saat dipanggil dari command retry)
     */
    public function sendResult(PaymentTransaction $tx, string $status, bool $recordFailure = true): bool
    {
        $payload = [
            'reference_id' => $tx->reference_id,
            'status'       => $status,            // paid | expired | failed
            'external_id'  => $tx->external_id,
            'provider'     => $tx->provider,
            'amount'       => (float) $tx->amount,
            'paid_at'      => $tx->paid_at?->toIso8601String(),
        ];

        $body      = json_encode($payload, JSON_UNESCAPED_SLASHES);
        $timestamp = now()->timestamp;
        $nonce     = bin2hex(random_bytes(16));
        $path      = (string) config('services.ovisito.callback_path');
        $signature = $this->signer->sign('POST', $path, $timestamp, $nonce, $body);

        try {
            // Kirim $body persis seperti yang di-hash (bukan array) agar signature konsisten
            $response = Http::withHeaders([
                    'X-Signature' => $signature,
                    'X-Timestamp' => (string) $timestamp,
                    'X-Nonce'     => $nonce,
                ])
                ->withBody($body, 'application/json')
                ->timeout(10)
                ->post(config('services.ovisito.base_url') . $path);

            if ($response->successful()) {
                $tx->update(['ovisito_synced_at' => now()]);
                Log::info('[Callback] Ovisito OK', [
                    'reference_id' => $tx->reference_id,
                    'status'       => $status,
                ]);
                return true;
            }

            Log::warning('[Callback] Ovisito non-2xx', [
                'reference_id' => $tx->reference_id,
                'http_status'  => $response->status(),
                'body'         => substr($response->body(), 0, 500),
            ]);
            if ($recordFailure) {
                $this->recordFailure($tx, $status, $response->status() . ': ' . substr($response->body(), 0, 500));
            }
            return false;

        } catch (\Throwable $e) {
            Log::error('[Callback] Ovisito exception', [
                'reference_id' => $tx->reference_id,
                'error'        => $e->getMessage(),
            ]);
            if ($recordFailure) {
                $this->recordFailure($tx, $status, $e->getMessage());
            }
            return false;
        }
    }

    private function recordFailure(PaymentTransaction $tx, string $status, string $error): void
    {
        DB::table('failed_callbacks')->insert([
            'transaction_uuid' => $tx->uuid,
            'payload'          => json_encode([
                'reference_id' => $tx->reference_id,
                'status'       => $status,
                'external_id'  => $tx->external_id,
                'provider'     => $tx->provider,
                'amount'       => (float) $tx->amount,
            ]),
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

## 11. Controller & Middleware

### 11.1 `VerifyPaymentApiKey`

`app/Http/Middleware/VerifyPaymentApiKey.php`

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class VerifyPaymentApiKey
{
    public function handle(Request $request, Closure $next)
    {
        $apiKey = (string) config('services.ovisito.api_key');
        $given  = (string) $request->header('X-API-KEY', '');

        if ($apiKey === '' || !hash_equals($apiKey, $given)) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        return $next($request);
    }
}
```

### 11.2 `PaymentController`

`app/Http/Controllers/Api/PaymentController.php`

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\PaymentTransaction;
use App\Services\PaymentManager;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class PaymentController extends Controller
{
    public function __construct(private readonly PaymentManager $manager) {}

    /** POST /api/payment/create */
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
            return response()->json($this->payload($existing, true));
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

            // Jangan bocorkan detail internal ke client saat production
            return response()->json([
                'error'   => 'Gateway error',
                'message' => config('app.debug') ? $e->getMessage() : 'Gagal membuat transaksi.',
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
            ],
        ]);

        return response()->json($this->payload($tx, false));
    }

    /** GET /api/payment/status/{ref} */
    public function status(string $ref)
    {
        $tx = PaymentTransaction::where('reference_id', $ref)->latest()->first();

        if (!$tx) {
            return response()->json(['error' => 'Not found'], 404);
        }

        return response()->json([
            'reference_id' => $tx->reference_id,
            'external_id'  => $tx->external_id,
            'provider'     => $tx->provider,
            'status'       => $tx->status,
            'amount'       => (float) $tx->amount,
            'paid_at'      => $tx->paid_at?->toIso8601String(),
            'synced'       => $tx->ovisito_synced_at !== null,
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

### 11.3 `WebhookController`

`app/Http/Controllers/Api/WebhookController.php`

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
            Log::warning('[Webhook] DANA tanpa external_id', ['raw' => $result['raw']]);
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
                return; // sudah diproses request paralel
            }

            $meta = array_merge($fresh->metadata ?? [], ['webhook_raw' => $result['raw']]);

            if ($status === 'paid') {
                $fresh->update(['status' => 'PAID', 'paid_at' => now(), 'metadata' => $meta]);
            } elseif (in_array($status, ['expired', 'failed'], true)) {
                $fresh->update(['status' => strtoupper($status), 'metadata' => $meta]);
            }
        });

        // Kirim callback DI LUAR transaksi DB agar lock tidak tertahan
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
                    'resultInfo' => [
                        'resultStatus' => 'S',
                        'resultCode'   => 'SUCCESS',
                        'resultMsg'    => 'OK',
                    ],
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
                    'resultInfo' => [
                        'resultStatus' => 'F',
                        'resultCode'   => $code,
                        'resultMsg'    => 'Error',
                    ],
                ],
            ],
        ], $httpStatus);
    }
}
```

### 11.4 `PaymentReturnController` & view

Halaman return **hanya menampilkan status** — bukan sumber kebenaran. Sumber kebenaran adalah webhook/query DANA.

`app/Http/Controllers/PaymentReturnController.php`

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

`resources/views/payment/return.blade.php`

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
            <p>E-ticket akan dikirim ke WhatsApp Anda dalam beberapa detik.</p>
        @elseif($tx->status === 'PENDING')
            <p>Pembayaran sedang diproses. Halaman ini refresh otomatis.</p>
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

## 12. Routing & Keamanan

### 12.1 `routes/api.php`

```php
<?php

use App\Http\Controllers\Api\PaymentController;
use App\Http\Controllers\Api\WebhookController;
use App\Http\Middleware\VerifyPaymentApiKey;
use Illuminate\Support\Facades\Route;

// Untuk Ovisito (dilindungi API key)
Route::middleware(VerifyPaymentApiKey::class)->prefix('payment')->group(function () {
    Route::post('/create',      [PaymentController::class, 'create']);
    Route::get('/status/{ref}', [PaymentController::class, 'status']);
});

// Untuk DANA (publik, diverifikasi signature)
Route::post('/webhook/dana', [WebhookController::class, 'dana']);
```

### 12.2 `routes/web.php`

```php
use App\Http\Controllers\PaymentReturnController;
use Illuminate\Support\Facades\Route;

Route::get('/payment/return', PaymentReturnController::class)->name('payment.return');
```

### 12.3 Verifikasi rute

```bash
php artisan route:list | grep -E "payment|webhook"
```

Hasil yang diharapkan:

```text
GET|HEAD  payment/return ............ payment.return
POST      api/payment/create ........ Api\PaymentController@create
GET|HEAD  api/payment/status/{ref} .. Api\PaymentController@status
POST      api/webhook/dana .......... Api\WebhookController@dana
```

### 12.4 CSRF

Rute `api/*` bawaan Laravel sudah berada di group `api` tanpa CSRF. Jika Anda memasangnya di `web.php`, kecualikan:

**Laravel 11+** (`bootstrap/app.php`):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: ['api/*']);
})
```

**Laravel 10** (`app/Http/Middleware/VerifyCsrfToken.php`):

```php
protected $except = ['api/*'];
```

> `GET /payment/return` tidak memerlukan pengecualian CSRF karena CSRF hanya berlaku pada method non-GET.

### 12.5 Hardening yang disarankan

| Kontrol | Implementasi |
|---|---|
| Rate limit endpoint create | `->middleware('throttle:60,1')` pada group `payment` |
| IP allowlist | Batasi `/api/payment/*` hanya ke IP server Ovisito (firewall/Nginx) |
| Webhook DANA | Allowlist IP DANA (bila disediakan) + verifikasi signature wajib |
| Log | `storage/logs` tidak world-readable; jangan log secret/private key |
| Transport | HTTPS wajib di production (termasuk callback ke Ovisito) |

---

## 13. Receiver Callback di Ovisito

Bagian ini dipasang di project **Ovisito** (`ovisitomcp.test`), bukan di `payment.test`.

### 13.1 Controller

`app/Http/Controllers/Api/InternalPaymentCallbackController.php`

Urutan verifikasi: header lengkap → timestamp → **signature** → baru **nonce** (agar penyerang tidak bisa "membakar" nonce dengan signature palsu).

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\transport_aceh\TransportBooking;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class InternalPaymentCallbackController extends Controller
{
    public function __invoke(Request $request)
    {
        // 1) Verifikasi HMAC
        $signature = (string) $request->header('X-Signature');
        $timestamp = (int)    $request->header('X-Timestamp');
        $nonce     = (string) $request->header('X-Nonce');

        if (!$signature || !$timestamp || !$nonce) {
            return response()->json(['error' => 'Missing signature headers'], 401);
        }

        if (abs(now()->timestamp - $timestamp) > 300) {
            return response()->json(['error' => 'Stale request'], 401);
        }

        $stringToSign = implode("\n", [
            'POST',
            '/api/internal/payment-callback',
            (string) $timestamp,
            $nonce,
            hash('sha256', $request->getContent()),
        ]);
        $expected = hash_hmac('sha256', $stringToSign, (string) config('services.payment.hmac_secret'));

        if (!hash_equals($expected, $signature)) {
            return response()->json(['error' => 'Invalid signature'], 401);
        }

        // Anti-replay: Cache::add atomik (false bila nonce sudah ada)
        if (!Cache::add('cb-nonce:' . $nonce, true, 600)) {
            return response()->json(['error' => 'Replay detected'], 401);
        }

        // 2) Validasi payload
        $data = $request->validate([
            'reference_id' => 'required|string',
            'status'       => 'required|in:paid,expired,failed',
            'external_id'  => 'required|string',
            'provider'     => 'required|string',
            'amount'       => 'required|numeric',
        ]);

        // 3) Proses
        DB::transaction(function () use ($data) {
            $booking = TransportBooking::on('transports')
                ->where('booking_code', $data['reference_id'])
                ->lockForUpdate()
                ->first();

            if (!$booking) {
                Log::warning('[Callback] Booking not found', $data);
                return;
            }

            if ($booking->status === TransportBooking::STATUS_PAID) {
                return; // idempotent
            }

            if ($data['status'] === 'paid') {
                // Eloquent update → Observer fire → WhatsApp terkirim
                $booking->update([
                    'status'                 => TransportBooking::STATUS_PAID,
                    'payment_provider'       => $data['provider'],
                    'payment_transaction_id' => $data['external_id'],
                ]);

                DB::connection('payments')->table('central_bookings')
                    ->where('booking_code', $data['reference_id'])
                    ->where('service_type', 'transport')
                    ->update([
                        'payment_status' => 'paid',
                        'status'         => 'confirmed',
                        'paid_at'        => now(),
                    ]);
            } else {
                $booking->update(['status' => 'cancelled']);
            }
        });

        return response()->json(['ok' => true]);
    }
}
```

> **Rekomendasi:** validasi juga bahwa `amount` pada callback sama dengan total booking sebelum menandai `PAID`.

### 13.2 Route, env, config di Ovisito

```php
// routes/api.php
Route::post('/internal/payment-callback',
    \App\Http\Controllers\Api\InternalPaymentCallbackController::class);
```

```env
PAYMENT_GATEWAY_URL=https://payment.ovisito.com
PAYMENT_API_KEY=<sama dengan payment.test>
PAYMENT_HMAC_SECRET=<sama dengan HMAC_SECRET di payment.test>
```

```php
// config/services.php
'payment' => [
    'url'         => env('PAYMENT_GATEWAY_URL'),
    'api_key'     => env('PAYMENT_API_KEY'),
    'hmac_secret' => env('PAYMENT_HMAC_SECRET'),
],
```

### 13.3 Contoh pemanggilan dari Ovisito

```php
$res = Http::withHeaders(['X-API-KEY' => config('services.payment.api_key')])
    ->timeout(15)
    ->post(config('services.payment.url') . '/api/payment/create', [
        'reference_id' => $booking->booking_code,
        'amount'       => $booking->total_amount,
        'provider'     => 'dana',
        'return_url'   => route('transport.payment.return', $booking->booking_code),
        'customer'     => ['name' => $booking->name, 'phone' => $booking->phone],
    ]);

return redirect()->away($res->json('checkout_url'));
```

---

## 14. Scheduler: Retry, Expire, Polling

### 14.1 `RetryFailedCallbacks`

`app/Console/Commands/RetryFailedCallbacks.php`

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

            $status = json_decode($row->payload, true)['status'] ?? strtolower($tx->status);

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
                $this->warn("✗ {$tx->reference_id} (attempt {$attempts})");
            }
        }

        return self::SUCCESS;
    }
}
```

### 14.2 `ExpirePendingTransactions`

`app/Console/Commands/ExpirePendingTransactions.php`

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

> Batas bawah 24 jam pada draf awal dihapus agar transaksi lama yang terlewat tetap ditandai.

### 14.3 `PollPendingTransactions`

`app/Console/Commands/PollPendingTransactions.php`

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

### 14.4 Penjadwalan

**Laravel 11+** (`routes/console.php`):

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('payment:retry-callbacks')->everyMinute()->withoutOverlapping();
Schedule::command('payment:expire-pending')->everyFiveMinutes()->withoutOverlapping();
Schedule::command('payment:poll-pending')->everyTwoMinutes()->withoutOverlapping();
```

**Laravel 10** (`app/Console/Kernel.php` → `schedule()`): gunakan tiga baris yang sama dengan `$schedule->command(...)`.

**Cron server:**

```bash
* * * * * cd /var/www/payment && php artisan schedule:run >> /dev/null 2>&1
```

---

## 15. Panduan Pengujian

> **Catatan:** dalam mode strict (`PAYMENT_REQUIRE_SIGNATURE=true`), webhook tanpa signature valid akan ditolak (401). Untuk uji manual via `curl` di lokal, set sementara `PAYMENT_REQUIRE_SIGNATURE=false`, lalu kembalikan ke `true` sebelum deploy.

| # | Skenario | Perintah / Langkah | Hasil yang diharapkan |
|---|---|---|---|
| 1 | Endpoint webhook hidup | `curl -X POST http://payment.test/api/webhook/dana -H "Content-Type: application/json" -d '{"response":{"body":{"acquirementId":"TEST","latestTransactionStatus":"SUCCESS"}}}'` | HTTP 200, `resultStatus: "S"` (non-strict) |
| 2 | Create payment | lihat §15.1 | JSON berisi `checkout_url` DANA. 401 = API key salah; 502 = konfigurasi DANA salah |
| 3 | Cashier tampil | Buka `checkout_url` di browser | Halaman DANA cashier sandbox muncul |
| 4 | Simulasi bayar sukses | Bayar di cashier sandbox | Webhook masuk → `PAID` → callback → WA terkirim |
| 5 | Idempotency | Kirim webhook sama 3× | Callback hanya 1×, `ovisito_synced_at` tidak berubah, WA 1× |
| 6 | Signature palsu | Kirim dengan `X-SIGNATURE: invalid` | HTTP 401, `resultStatus: "F"` |
| 7 | Expire | Buat transaksi, tunggu lewat `expires_at`, `php artisan payment:expire-pending` | Status `EXPIRED`, callback `expired` terkirim |
| 8 | Retry callback | Matikan Ovisito sebentar, bayar, nyalakan lagi, `php artisan payment:retry-callbacks` | Satu baris `failed_callbacks` → `resolved_at` terisi |

### 15.1 Contoh create payment

```bash
curl -X POST http://payment.test/api/payment/create \
  -H "X-API-KEY: <PAYMENT_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "reference_id": "TRX-TEST-001",
    "amount": 150000,
    "provider": "dana",
    "return_url": "http://payment.test/payment/return",
    "customer": {"name":"Budi","phone":"081234567890","email":"budi@test.com"}
  }'
```

### 15.2 Verifikasi hasil

```bash
tail -f storage/logs/laravel.log | grep -iE "webhook|callback|dana"
```

```sql
-- payment.test
SELECT reference_id, status, paid_at, ovisito_synced_at
FROM payment_transactions WHERE reference_id = 'TRX-TEST-001';
-- Harapan: PAID, paid_at & ovisito_synced_at terisi

-- Ovisito
SELECT booking_code, status, payment_provider, wa_sent_at
FROM transport_bookings WHERE booking_code = 'TRX-TEST-001';
-- Harapan: paid, dana, wa_sent_at terisi
```

---

## 16. Deployment Checklist

### 16.1 Sebelum deploy

| # | Item | ☐ |
|---|---|---|
| 1 | `.env` lengkap (DANA + Ovisito) | ☐ |
| 2 | `DANA_ORDER_NOTIFY_URL` & `DANA_PAY_RETURN_URL` = HTTPS publik | ☐ |
| 3 | `DANA_PUB_KEY` valid (tidak kosong) | ☐ |
| 4 | `PAYMENT_REQUIRE_SIGNATURE=true` | ☐ |
| 5 | `PAYMENT_API_KEY` & `HMAC_SECRET` identik di kedua project | ☐ |
| 6 | `php artisan migrate --force` sudah dijalankan | ☐ |
| 7 | `php artisan route:list` menampilkan 4 rute inti | ☐ |
| 8 | Cron `schedule:run` aktif (`crontab -l`) | ☐ |
| 9 | `storage/logs` tidak world-readable | ☐ |
| 10 | `APP_ENV=production`, `APP_DEBUG=false` | ☐ |
| 11 | Credential DANA production (bukan sandbox) sudah diganti | ☐ |
| 12 | Rate limit & IP allowlist aktif (§12.5) | ☐ |

### 16.2 Setelah deploy

```bash
php artisan config:cache
php artisan route:cache

# Health check: API key valid
curl -X POST https://payment.ovisito.com/api/payment/create \
  -H "X-API-KEY: <key>" -H "Content-Type: application/json" \
  -d '{"reference_id":"TRX-DEPLOY-TEST","amount":10000,"provider":"dana","return_url":"https://payment.ovisito.com/payment/return"}'

# Bayar di sandbox/production kecil → cek log & DB
```

---

## 17. Menambah Gateway Baru

Contoh: Flip, GoPay, BSI VA.

**1. Buat class gateway** — `app/Services/Gateways/FlipGateway.php`:

```php
class FlipGateway implements PaymentGatewayInterface
{
    public function getName(): string { return 'flip'; }

    public function createTransaction(array $data): array
    {
        // Panggil API Flip → return ['external_id', 'checkout_url', 'expires_at']
    }

    public function handleWebhook(Request $request): array
    {
        // Verifikasi signature Flip → return ['status', 'reference_id', 'external_id', 'amount', 'raw']
    }
}
```

**2. Daftarkan di `PaymentManager`:**

```php
'flip'  => new FlipGateway(),
'gopay' => new GopayGateway(),
```

**3. Tambah route webhook:**

```php
Route::post('/webhook/flip', [WebhookController::class, 'flip']);
```

**4. Refactor `WebhookController`:** pindahkan logika `dana()` ke `handleGenericWebhook(Request $request, string $provider)` lalu panggil dari tiap method provider:

```php
public function flip(Request $request) { return $this->handleGenericWebhook($request, 'flip'); }
```

Tidak ada perubahan pada `PaymentController`, `OvisitoCallbackService`, maupun skema database.

---

## 18. Troubleshooting

| Gejala | Kemungkinan penyebab | Tindakan |
|---|---|---|
| `401 Unauthorized` saat create | `X-API-KEY` salah / `PAYMENT_API_KEY` kosong | Samakan key, `php artisan config:clear` |
| `502 Gateway error` | Kredensial/private key DANA salah, format `\n` pada key | Periksa `.env`, lihat `[DANA] createOrder` di log |
| Webhook tidak masuk | URL notify bukan HTTPS publik / tunnel mati | Cek ngrok, cek `DANA_ORDER_NOTIFY_URL` |
| Webhook 401 | Signature gagal / `DANA_PUB_KEY` kosong (strict) | Isi public key DANA; kirim contoh header `X-SIGNATURE` untuk dianalisis |
| Callback ke Ovisito 401 | `HMAC_SECRET` beda, jam server selisih >5 menit, path berbeda | Samakan secret, sinkronkan NTP, cocokkan `callback_path` |
| `failed_callbacks` menumpuk | Ovisito down / validasi gagal | Cek log Ovisito, jalankan `payment:retry-callbacks` |
| Nominal salah di cashier | Satuan `orderAmount.value` | Lihat catatan di bawah |

**Catatan penting yang perlu diverifikasi terhadap dokumentasi DANA resmi sebelum production:**

1. **Satuan nominal** — beberapa API DANA memakai satuan sen (mis. Rp 1.000 → `100000`). Konfirmasi di sandbox dengan transaksi kecil, lalu sesuaikan konversi di `DanaGateway::createTransaction()` dan pembacaan `orderAmount.value` pada webhook.
2. **Format signature webhook** — draf ini memverifikasi RSA-SHA256 atas raw body. Cocokkan dengan *string-to-sign* yang ditetapkan DANA untuk versi API yang Anda pakai.
3. **Nama method package** — pastikan `DANAPay::createOrder()` dan `queryOrder()` sesuai versi `otnansirk/laravel-dana` yang terpasang.

---

## 19. Catatan Perubahan dari Draf Awal

| # | Perbaikan | Alasan |
|---|---|---|
| 1 | Binding `HmacSigner` di service provider (§5.4) | Tanpa binding, container tidak bisa membuat `OvisitoCallbackService` (secret string tidak bisa di-autowire). |
| 2 | Mode `PAYMENT_REQUIRE_SIGNATURE` (strict default) | Draf awal menerima webhook tanpa signature — celah pemalsuan status pembayaran. |
| 3 | Callback dikirim sebagai raw body (`withBody`) | Memastikan hash body pada signature identik dengan yang diterima Ovisito. |
| 4 | `recordFailure` opsional | Command retry sebelumnya menambah baris `failed_callbacks` baru di setiap kegagalan. |
| 5 | Status callback pada `failed_callbacks` disimpan eksplisit | Sebelumnya status `EXPIRED/FAILED` bisa salah dipetakan saat retry. |
| 6 | Nonce divalidasi setelah signature, memakai `Cache::add` | Mencegah nonce-poisoning dan race condition replay. |
| 7 | Unique index `(provider, external_id)` | Menjamin satu transaksi gateway tidak terduplikasi. |
| 8 | `reference_id` disisipkan ke return URL | DANA tidak selalu mengirim parameter yang bisa dipakai halaman return. |
| 9 | `<meta refresh>` dipindah ke `<head>` | Valid HTML dan perilaku konsisten di browser. |
| 10 | Pesan error gateway disembunyikan saat `APP_DEBUG=false` | Mencegah kebocoran detail internal. |
| 11 | Batas 24 jam pada expire dihapus | Transaksi lama yang terlewat tetap ditutup. |
| 12 | Kredensial contoh diganti placeholder | Menghindari nilai yang menyerupai secret asli. |
| 13 | `needsCallback()` mencakup `EXPIRED/FAILED` | Callback non-paid ikut dijamin tersinkron. |

---

## 20. Rencana Eksekusi

| Hari | Fokus | Kriteria selesai |
|---|---|---|
| **1** | Pasang file §4–§12, migrate, `route:list`, Test 1–2 | `checkout_url` DANA muncul |
| **2** | Setup tunnel, set notify URL, Test 3–4 | Webhook masuk, transaksi `PAID` |
| **3** | Receiver di Ovisito (§13), uji end-to-end | Bayar DANA → callback → WA terkirim |
| **4** | Scheduler (§14), Test 5–8, hardening §12.5 | Retry/expire/polling berjalan |
| **5+** | Deploy production (§16), lalu extend gateway (§17) | Checklist §16 lengkap |

**Estimasi total:** ± 2,5 jam pemasangan kode + waktu pengujian sandbox.

---

*Dokumen ini hidup — perbarui versi dan catatan perubahan setiap ada modifikasi arsitektur atau penambahan gateway.*
