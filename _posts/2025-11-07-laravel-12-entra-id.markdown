---
layout: post 
title: "Panduan Integrasi Pengesahan Microsoft Entra ID dalam Laravel 12 Menggunakan Socialite" 
date: 2025-11-07 01:29:00 +0800 
categories: [tutorial, programming] 
tags: [laravel, laravel 12, php, socialite, microsoft, entra id, azure, authentication] 
lang: ms-MY
---
Baru-baru ini, satu pertanyaan telah diterima berkenaan pilihan alternatif selain daripada penggunaan `Active Directory (AD)` tempatan untuk pengesahan LDAP. Terdapat kerisauan sekiranya pelayan AD tempatan mengalami kegagalan (downtime), ia akan menjejaskan keupayaan pengguna untuk mengakses sistem-sistem yang berkaitan.

Ini adalah satu kebimbangan yang sah. Ketergantungan sepenuhnya kepada infrastruktur AD tempatan boleh mendedahkan organisasi kepada risiko kegagalan di mana isu Ketersediaan Tinggi (High Availability - HA) menjadi tidak terjamin. Memandangkan banyak organisasi telahpun mengguna pakai Microsoft Entra ID, satu cadangan penyelesaian yang lebih optimum adalah dengan memanfaatkan platform tersebut. Pendekatan ini mengalihkan proses pengesahan ke persekitaran awan (cloud). Selagi aplikasi mempunyai capaian internet, pengguna boleh membuat pengesahan seperti biasa, walaupun pelayan AD tempatan sedang mengalami gangguan.

Oleh itu, tutorial ini disediakan sebagai jawapannya. Kami akan menunjukkan panduan langkah demi langkah untuk mengintegrasikan log masuk menggunakan Entra ID dalam aplikasi Laravel 12.

Panduan ini akan menunjukkan cara membina aplikasi web Laravel 12 bermula dari aplikasi kosong, dengan matlamat untuk mengintegrasikan sistem pengesahan pengguna menggunakan Microsoft Entra ID (dahulunya dikenali sebagai Azure AD). Kita akan memanfaatkan pakej Laravel Socialite bersama-sama pembekal (provider) `SocialiteProviders/microsoft`.

Proses ini telah dipecahkan kepada 10 langkah utama untuk memudahkan rujukan.

## Langkah 1: Konfigurasi Persekitaran Pembangunan

Pertama sekali, adalah penting untuk memastikan persekitaran pembangunan anda memenuhi semua prasyarat yang diperlukan untuk Laravel 12.

1. Prasyarat:
* PHP: Versi 8.2 atau lebih tinggi. Anda boleh mengesahkan versi melalui arahan php -v.
* Composer: Diperlukan untuk pengurusan pakej PHP dan pemasangan Laravel serta pakej-pakej kebergantungan.
* Node.js: Diperlukan untuk proses kompilasi (build) aset frontend Laravel.
2. Pemasangan Laravel 12:
Buka terminal anda dan laksanakan arahan Composer berikut untuk mencipta projek baharu dengan nama entra-app.
``` bash
composer create-project laravel/laravel entra-app
``` 
3. Masuk ke Direktori Projek:
Setelah proses pemasangan selesai, navigasi ke direktori projek yang baru dicipta.

``` bash
cd entra-app
```

## Langkah 2: Pendaftaran Aplikasi di Portal Azure

Sebelum memulakan pengekodan, aplikasi anda perlu didaftarkan terlebih dahulu di dalam Microsoft Entra ID. Proses ini bertujuan untuk mendaftarkan aplikasi anda secara rasmi dengan platform identiti Microsoft dan mendapatkan kredensial yang diperlukan.

1. Log masuk ke pusat pentadbir [Microsoft Entra](https://entra.microsoft.com/).
2. Navigasi ke: Entra ID > App registrations.
3. Klik + New registration.
4. Lengkapkan borang pendaftaran:
* Name: My Laravel App (atau sebarang nama yang deskriptif).
* Supported account types: Untuk tutorial ini, yang memfokuskan kepada akaun perniagaan Entra ID, sila pilih "Accounts in this organizational directory only (Single tenant)".
	- (Nota: Pilihan "Accounts in any organizational directory... and personal Microsoft accounts..." akan membenarkan log masuk dari pelbagai organisasi dan akaun peribadi Microsoft.)
* Redirect URI:
	- Pilih platform Web.
	- Masukkan URL berikut: `http://localhost:8000/auth/microsoft/callback`
	- URL ini mesti sepadan dengan definisi laluan (route) callback yang akan dikonfigurasi dalam Langkah 7.
* Klik Register.
* Perolehi Kredensial Aplikasi:
Pada halaman Overview aplikasi, salin dan simpan dua nilai berikut di lokasi yang selamat:
	- Application (client) ID
	- Directory (tenant) ID
* Cipta Client Secret:
	- Pada menu navigasi kiri, klik Certificates & secrets.
	- Pilih tab Client secrets.
	- Klik + New client secret.
	- Berikan deskripsi (cth: laravel-app-secret) dan pilih tempoh sah.
	- Klik Add.
	- MAKLUMAT PENTING: Salin nilai (Value) secret yang dijana dengan segera. Nilai ini hanya akan dipaparkan sekali sahaja dan tidak boleh diperoleh semula selepas anda meninggalkan halaman ini.

Pada tahap ini, anda sepatutnya mempunyai tiga maklumat penting: `Client ID`, `Tenant ID`, dan `Client Secret`.

## Langkah 3: Konfigurasi Pangkalan Data dan Fail Persekitaran (.env)

Seterusnya, kita akan mengkonfigurasi aplikasi Laravel untuk berhubung dengan pangkalan data dan menyimpan kredensial Microsoft yang telah diperoleh.

1. Konfigurasi Fail `.env`:
Buka fail `.env` yang terletak di direktori akar projek anda.
2. Pangkalan Data:
Untuk tujuan pembangunan dan demonstrasi ini, SQLite akan digunakan bagi memudahkan proses. Cipta fail pangkalan data terlebih dahulu:

``` bash
touch database/database.sqlite
```

Kemudian, kemas kini konfigurasi `DB_` di dalam fail `.env`:

``` dotenv
DB_CONNECTION=sqlite
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=laravel
# DB_USERNAME=root
# DB_PASSWORD=
```

(Sekiranya MySQL menjadi pilihan, sila lengkapkan butiran sambungan pangkalan data anda seperti biasa.)

Konfigurasi Session Driver:
Ini adalah langkah kritikal untuk mengelakkan ralat InvalidStateException semasa proses OAuth2. Kita akan mengkonfigurasi Laravel untuk menyimpan sesi di dalam pangkalan data.

Di dalam fail `.env`, cari `SESSION_DRIVER` dan `SESSION_DOMAIN`, kemudian kemas kini nilainya:

``` dotenv
SESSION_DRIVER=database
SESSION_DOMAIN=localhost
```

Nota: Penetapan `SESSION_DOMAIN=localhost` adalah amat penting semasa pembangunan di persekitaran tempatan.

Kredensial Perkhidmatan Microsoft:
Tambahkan tiga kredensial yang diperoleh dari Langkah 2 ke bahagian akhir fail `.env`:

``` dotenv
MICROSOFT_CLIENT_ID="YOUR_APPLICATION_CLIENT_ID"
MICROSOFT_CLIENT_SECRET="YOUR_CLIENT_SECRET_VALUE"
MICROSOFT_TENANT_ID="YOUR_DIRECTORY_TENANT_ID"
MICROSOFT_REDIRECT_URI="http://localhost:8000/auth/microsoft/callback"
```

## Langkah 4: Pemasangan Pakej Socialite

Langkah ini melibatkan pemasangan pakej Laravel Socialite dan provider Microsoft yang diperlukan melalui Composer.

1. Pasang Pakej Asas Laravel Socialite:

``` bash
composer require laravel/socialite
```

2. Pasang Pakej Provider Microsoft:

``` bash
composer require socialiteproviders/microsoft
```

## Langkah 5: Konfigurasi Pembekal (Provider) Socialite

Setelah pakej dipasang, ia perlu didaftarkan dan dikonfigurasi di dalam aplikasi Laravel.

Daftar Konfigurasi Servis:
Buka fail `config/services.php`. Tambahkan tatasusunan (array) berikut ke dalam fail tersebut:

``` php
'microsoft' => [
    'client_id' => env('MICROSOFT_CLIENT_ID'),
    'client_secret' => env('MICROSOFT_CLIENT_SECRET'),
    'redirect' => env('MICROSOFT_REDIRECT_URI'),
    'tenant' => env('MICROSOFT_TENANT_ID'),
],
```

Daftar Event Listener:
Bermula Laravel 11, `EventServiceProvider` tidak lagi didaftarkan secara lalai. Oleh itu, kita akan mendaftarkan listener untuk Socialite di dalam `AppServiceProvider`.

Buka `app/Providers/AppServiceProvider.php`.

Tambahkan dua pernyataan use berikut di bahagian atas fail:

``` php
use Illuminate\Support\Facades\Event;
use SocialiteProviders\Manager\SocialiteWasCalled;
```

Kemudian, di dalam kaedah (method) `boot()`, tambahkan kod berikut:

``` php
public function boot(): void
{
    Event::listen(function (SocialiteWasCalled $event) {
        $event->extendSocialite('microsoft', \SocialiteProviders\Microsoft\Provider::class);
    });
}
```

Konfigurasi ini memastikan Laravel kini mengenali `microsoft` sebagai salah satu provider Socialite.

## Langkah 6: Pengubahsuaian Jadual Pengguna (Users)

Data pengguna yang diterima daripada Microsoft perlu disimpan di dalam pangkalan data kita. Jadual users sedia ada perlu diubah suai untuk menampung maklumat ini.

Kemas kini Fail Migrasi `create_users_table`:
Buka fail migrasi pengguna di database/migrations/ (cth: ..._create_users_table.php).

Ubah suai kaedah `up()` untuk menambah kolum `microsoft_id` (untuk menyimpan ID unik Microsoft), `avatar` (untuk pautan imej profil), dan jadikan kolum `password` sebagai nullable, kerana pengesahan akan diuruskan oleh Microsoft.

Blok `Schema::create('users', ...)` anda sepatutnya kelihatan seperti ini:

``` php
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->timestamp('email_verified_at')->nullable();
    $table->string('password')->nullable(); // Jadikan kata laluan nullable
    $table->string('microsoft_id')->nullable()->unique(); // Tambah ini
    $table->string('avatar')->nullable(); // Tambah ini
    $table->rememberToken();
    $table->timestamps();
});
```

Cipta Migrasi Jadual Sesi:
Seperti yang dikonfigurasi dalam Langkah 3 (`SESSION_DRIVER=database`), kita perlu menjana fail migrasi untuk jadual sesi.

``` bash
php artisan session:table
```

Laksanakan Migrasi:
Laksanakan arahan berikut untuk mencipta jadual users (yang telah diubah suai) dan jadual sessions di dalam pangkalan data.

``` bash
php artisan migrate
```

Kemas kini Model User:
Buka fail `app/Models/User.php`. Tambahkan `microsoft_id` dan `avatar` ke dalam senarai `$fillable` untuk membenarkan mass assignment.

``` php
protected $fillable = [
    'name',
    'email',
    'password',
    'microsoft_id', // Tambah ini
    'avatar',       // Tambah ini
];
```

## Langkah 7: Definisi Laluan (Routes) Pengesahan

Laluan-laluan web berikut perlu didefinisikan untuk menguruskan aliran pengesahan OAuth2.

Buka fail `routes/web.php` dan tambahkan definisi laluan berikut:

``` php
use App\Http\Controllers\MicrosoftAuthController;

// Halaman Utama
Route::get('/', function () {
    return view('welcome');
});

// Laluan Pengesahan Microsoft
Route::get('/auth/microsoft/redirect', [MicrosoftAuthController::class, 'redirect'])
    ->name('microsoft.redirect');

Route::get('/auth/microsoft/callback', [MicrosoftAuthController::class, 'callback'])
    ->name('microsoft.callback');

// Laluan dashboard yang dilindungi (memerlukan pengesahan)
Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware('auth')->name('dashboard');

// Laluan Log Keluar
Route::post('/logout', [MicrosoftAuthController::class, 'logout'])
    ->middleware('auth')->name('logout');
```

## Langkah 8: Implementasi Logik Controller

Ini adalah komponen teras yang akan menguruskan logik untuk mengalih pengguna ke Microsoft dan memproses callback selepas pengesahan berjaya.

Cipta Controller:
Jana fail controller baharu menggunakan Artisan.

``` bash
php artisan make:controller MicrosoftAuthController
```

Implementasi Logik:
Buka fail yang baru dicipta di `app/Http/Controllers/MicrosoftAuthController.php`. Gantikan keseluruhan kandungan fail dengan kod berikut. Komen di dalam kod menerangkan fungsi setiap kaedah.

``` php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Laravel\Socialite\Facades\Socialite;

class MicrosoftAuthController extends Controller
{
    /**
     * Alihkan pengguna ke halaman pengesahan Microsoft.
     */
    public function redirect()
    {
        // Skop 'openid', 'profile', dan 'email' adalah skop umum
        // yang diperlukan untuk mendapatkan maklumat asas pengguna.
        return Socialite::driver('microsoft')
            ->scopes(['openid', 'profile', 'email'])
            ->redirect();
    }

    /**
     * Dapatkan maklumat pengguna dari Microsoft selepas pengesahan.
     */
    public function callback()
    {
        // Dapatkan objek pengguna daripada Microsoft Socialite
        $microsoftUser = Socialite::driver('microsoft')->user();

        // Gunakan updateOrCreate untuk mencari pengguna sedia ada
        // berdasarkan microsoft_id, atau cipta pengguna baru jika tidak ditemui.
        $user = User::updateOrCreate(
            [
                'microsoft_id' => $microsoftUser->getId(), // Kunci carian
            ],
            [
                'name' => $microsoftUser->getName(),
                'email' => $microsoftUser->getEmail(),
                'avatar' => $microsoftUser->getAvatar(),
                'password' => Hash::make(str()->random(24)) // Cipta kata laluan rawak
            ]
        );

        // Log masuk pengguna ke dalam aplikasi
        Auth::login($user);

        // Alihkan pengguna ke dashboard
        return redirect()->route('dashboard');
    }

    /**
     * Log keluar pengguna dari aplikasi.
     */
    public function logout(Request $request)
    {
        Auth::logout();

        $request->session()->invalidate();
        $request->session()->regenerateToken();

        return redirect('/');
    }
}
```

## Langkah 9: Pembangunan Antara Muka Pengguna (Views)

Kita hampir ke penghujung. Langkah ini adalah untuk menyediakan antara muka pengguna yang ringkas bagi membolehkan proses pengujian.

Pemasangan Aset Frontend:
Laksanakan arahan berikut untuk memasang pakej Node.js dan memulakan pelayan Vite untuk kompilasi aset.

``` bash
npm install
npm run dev
```

(Biarkan proses `npm run dev` ini berjalan di terminal.)

Cipta View Dashboard:
Cipta fail baru di `resources/views/dashboard.blade.php`:

``` html
<!DOCTYPE html>
<html lang="ms-MY">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Dashboard</title>
    @vite('resources/css/app.css')
</head>
<body class="bg-gray-100 font-sans">
    <div class="min-h-screen flex flex-col items-center justify-center">
        <div class="bg-white p-8 rounded-lg shadow-md w-full max-w-md">
            <h1 class="text-2xl font-bold mb-4">Selamat Datang ke Dashboard Anda!</h1>

            @auth
                <div class="flex items-center space-x-4 mb-6">
                    @if(Auth::user()->avatar)
                        <img src="{{ Auth::user()->avatar }}" alt="Avatar" class="w-16 h-16 rounded-full">
                    @endif
                    <div>
                        <p class="text-lg font-semibold">{{ Auth::user()->name }}</p>
                        <p class="text-gray-600">{{ Auth::user()->email }}</p>
                    </div>
                </div>

                <form action="{{ route('logout') }}" method="POST">
                    @csrf
                    <button type="submit" class="w-full bg-red-500 text-white py-2 rounded-lg hover:bg-red-600 transition">
                        Log Keluar
                    </button>
                </form>
            @endauth
        </div>
    </div>
</body>
</html>
```

Kemas kini View Welcome:
Buka `resources/views/welcome.blade.php`. Gantikan keseluruhan kandungannya dengan kod berikut untuk halaman log masuk:

``` html
<!DOCTYPE html>
<html lang="ms-MY">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Log Masuk</title>
    @vite('resources/css/app.css')
</head>
<body class="bg-gray-100 font-sans">
    <div class="min-h-screen flex items-center justify-center">
        <div class="bg-white p-8 rounded-lg shadow-md w-full max-w-sm">

            @auth
                <h2 class="text-xl font-semibold text-center mb-4">Anda telah log masuk!</h2>
                <a href="{{ route('dashboard') }}" class="block w-full text-center bg-blue-500 text-white py-2 rounded-lg hover:bg-blue-600 transition mb-2">
                    Pergi ke Dashboard
                </a>
                <form action="{{ route('logout') }}" method="POST">
                    @csrf
                    <button type="submit" class="w-full bg-gray-300 text-gray-700 py-2 rounded-lg hover:bg-gray-400 transition">
                        Log Keluar
                    </button>
                </form>
            @else
                <h1 class="text-2xl font-bold text-center mb-6">Log Masuk ke Aplikasi Anda</h1>
                <a href="{{ route('microsoft.redirect') }}" class="flex items-center justify-center w-full bg-blue-600 text-white py-2 px-4 rounded-lg hover:bg-blue-700 transition">
                    <!-- SVG Logo Microsoft Ringkas -->
                    <svg classs="w-5 h-5 mr-2" fill="currentColor" viewBox="0 0 23 23">
                        <path d="M0 0H11V11H0V0ZM12 0H23V11H12V0ZM0 12H11V23H0V12ZM12 12H23V23H12V12Z"/>
                    </svg>
                    Log masuk dengan Microsoft
                </a>
            @endauth

        </div>
    </div>
</body>
</html>
```

## Langkah 10: Pengujian Penuh Aplikasi

Aplikasi kini sedia untuk diuji.

1. Mulakan Pelayan Pembangunan:
Di dalam terminal anda (pastikan npm run dev masih berjalan di terminal berasingan), mulakan pelayan Laravel Artisan:

``` bash
php artisan serve
```

2. Proses Ujian Aliran:

* Buka pelayar web anda dan navigasi ke `http://localhost:8000`.
* Anda akan dipaparkan dengan halaman "Log Masuk".
* Klik pada butang "Log masuk dengan Microsoft".
* Anda akan dialihkan ke halaman log masuk Microsoft.
* Laksanakan pengesahan menggunakan akaun Entra ID (Business) anda.
* Setelah berjaya, anda akan dialihkan kembali ke aplikasi anda, terus ke halaman `/dashboard`. Maklumat nama dan emel anda sepatutnya dipaparkan.
* Uji fungsi "Log Keluar". Anda sepatutnya dikembalikan ke halaman log masuk utama.

Jika aliran ini berfungsi seperti yang diterangkan, anda telah berjaya mengintegrasikan pengesahan Microsoft Entra ID ke dalam aplikasi Laravel 12 anda.

### Rujukan Kod Sumber

Untuk rujukan penuh kod sumber bagi projek demo ini, anda boleh mengakses repositori GitHub di:
[https://github.com/apisznasdin/entra-app/](https://github.com/apisznasdin/entra-app/)

![Entra App Login]({{ "assets/images/entra-app-login.png" | relative_url }})
![Entra App Login]({{ "assets/images/entra-app-sso.png" | relative_url }})
![Entra App Login]({{ "assets/images/entra-app-dashboard.png" | relative_url }})