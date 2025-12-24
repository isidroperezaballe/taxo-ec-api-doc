# 🚀 Setup de Queue Workers para Sincronización de Buzón Tributario

## Fecha: 6 de Diciembre, 2025

---

## 📊 **CONTEXTO**

Con **3025+ taxpayers**, la sincronización directa (secuencial) tarda ~50 minutos.  
Con **Jobs/Queues** (paralelo con 10 workers), tarda ~5 minutos. ⚡ **10x más rápido**

---

## 🎯 **ARQUITECTURA**

### Componentes Creados

```
app/Jobs/SyncTaxPayerMailboxJob.php          ← Job individual por taxpayer
app/Console/Commands/SyncTaxMailboxCommand.php  ← Comando con opción --queue
```

### Flujo de Ejecución

```
1. Comando despacha 3025 jobs a la cola "mailbox-sync"
   ↓
2. Queue workers (10 en paralelo) toman jobs de la cola
   ↓
3. Cada worker ejecuta SyncTaxPayerMailboxJob
   ↓
4. Si falla → Reintenta (3 veces con backoff)
   ↓
5. Si falla 3 veces → Se marca como failed y se loggea
```

---

## 📋 **SETUP PASO A PASO**

### 1️⃣ **Configurar Queue Connection**

Tienes **3 opciones** (de más simple a más robusto):

#### **Opción A: Database Queue (Recomendado para empezar)**

**Ventajas:**
- ✅ No requiere Redis ni servicios externos
- ✅ Usa tu MySQL actual
- ✅ Setup inmediato (2 minutos)

**Setup:**

```bash
# 1. Crear tablas de queue
php artisan queue:table
php artisan queue:failed-table
php artisan migrate

# 2. Configurar .env
QUEUE_CONNECTION=database
```

---

#### **Opción B: Redis Queue (Recomendado para producción)**

**Ventajas:**
- ✅ Más rápido que database
- ✅ Menor carga en MySQL
- ✅ Escalable

**Setup:**

```bash
# 1. Instalar Redis (si no lo tienes)
sudo apt-get install redis-server
sudo systemctl start redis
sudo systemctl enable redis

# 2. Instalar extensión PHP Redis
sudo apt-get install php-redis
# O con pecl:
sudo pecl install redis
# Agregar a php.ini: extension=redis.so

# 3. Instalar paquete Laravel
composer require predis/predis

# 4. Configurar .env
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# 5. Crear tabla de failed jobs
php artisan queue:failed-table
php artisan migrate
```

---

#### **Opción C: Sync (Solo para testing, NO usar en producción)**

```bash
# .env
QUEUE_CONNECTION=sync
```

**⚠️ Esto ejecuta jobs de forma secuencial (sin paralelismo)**

---

### 2️⃣ **Ejecutar Queue Workers**

Una vez configurado el queue connection, necesitas **workers corriendo 24/7**:

```bash
# Worker básico (1 worker)
php artisan queue:work --queue=mailbox-sync --tries=3 --timeout=120

# Múltiples workers (10 workers en paralelo)
# Necesitas ejecutar este comando 10 veces (en 10 terminales o con supervisor)
php artisan queue:work --queue=mailbox-sync --tries=3 --timeout=120 &
```

**Parámetros:**
- `--queue=mailbox-sync` → Cola específica del buzón tributario
- `--tries=3` → Reintentar hasta 3 veces si falla
- `--timeout=120` → Timeout de 120 segundos por job

---

### 3️⃣ **Mantener Workers Corriendo (Supervisor)**

Los queue workers deben correr **24/7**. Si se caen, los jobs no se procesarán.

**Solución: Usar Supervisor** (proceso daemon que reinicia workers automáticamente)

#### **Instalar Supervisor**

```bash
sudo apt-get install supervisor
```

#### **Configurar Worker para Mailbox Sync**

Crear archivo: `/etc/supervisor/conf.d/taxo-mailbox-worker.conf`

```ini
[program:taxo-mailbox-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /home/isidro/code/taxo-ec/artisan queue:work database --queue=mailbox-sync --tries=3 --timeout=120 --max-jobs=1000 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=isidro
numprocs=10
redirect_stderr=true
stdout_logfile=/home/isidro/code/taxo-ec/storage/logs/worker-mailbox.log
stopwaitsecs=3600
```

**Explicación:**
- `numprocs=10` → 10 workers en paralelo
- `max-jobs=1000` → Reiniciar worker cada 1000 jobs (evita memory leaks)
- `max-time=3600` → Reiniciar worker cada 1 hora
- `user=isidro` → Usuario que ejecuta los workers
- `autostart=true` → Iniciar automáticamente al bootear el servidor
- `autorestart=true` → Reiniciar automáticamente si se cae

#### **Activar Supervisor**

```bash
# Recargar configuración
sudo supervisorctl reread
sudo supervisorctl update

# Iniciar workers
sudo supervisorctl start taxo-mailbox-worker:*

# Ver estado
sudo supervisorctl status

# Logs en tiempo real
tail -f /home/isidro/code/taxo-ec/storage/logs/worker-mailbox.log
```

---

### 4️⃣ **Ejecutar Sincronización con Jobs**

```bash
# Con flag --queue para usar Jobs (RECOMENDADO para 3025 taxpayers)
php artisan app:sync-tax-mailbox --queue

# Sin flag (modo directo/secuencial) - Solo para testing o <500 taxpayers
php artisan app:sync-tax-mailbox
```

**Salida esperada con --queue:**

```
🚀 Despachando jobs de sincronización a la cola...
💡 Los jobs se procesarán en paralelo por los queue workers

📊 Total de contribuyentes: 3025

 3025/3025 [████████████████████████████] 100% - Completado

✅ Jobs despachados exitosamente a la cola

┌──────────────────────────────┬─────────────────┐
│ Métrica                      │ Valor           │
├──────────────────────────────┼─────────────────┤
│ Total de jobs despachados    │ 3025            │
│ Cola                         │ mailbox-sync    │
│ Reintentos por job           │ 3               │
│ Timeout por job              │ 120 segundos    │
│ Backoff entre reintentos     │ 60s, 120s, 300s │
└──────────────────────────────┴─────────────────┘

📊 Monitoreo:
   • Laravel Horizon: http://taxo-ec.test/horizon
   • Logs: tail -f storage/logs/laravel.log | grep "mailbox"
   • Queue status: php artisan queue:monitor mailbox-sync

⏱️  Tiempo estimado de procesamiento:
   Con 10 workers: ~5 minutos (vs ~51 minutos en modo directo)
```

---

## 🔍 **MONITOREO Y DEBUGGING**

### Ver Jobs en la Cola

```bash
# Ver estadísticas de la cola
php artisan queue:monitor mailbox-sync

# Ver failed jobs
php artisan queue:failed

# Reintentar un failed job específico
php artisan queue:retry [job_id]

# Reintentar TODOS los failed jobs
php artisan queue:retry all

# Ver tabla de jobs (si usas database queue)
mysql -u root -p taxo_ec -e "SELECT * FROM jobs WHERE queue = 'mailbox-sync' LIMIT 10;"

# Ver failed jobs en DB
mysql -u root -p taxo_ec -e "SELECT * FROM failed_jobs ORDER BY failed_at DESC LIMIT 10;"
```

### Logs

```bash
# Ver logs de workers
tail -f storage/logs/worker-mailbox.log

# Ver logs de Laravel (filtra por "mailbox")
tail -f storage/logs/laravel.log | grep -i mailbox

# Ver logs de Supervisor
sudo tail -f /var/log/supervisor/supervisord.log
```

### Laravel Horizon (Opcional pero recomendado)

Horizon es una UI web hermosa para monitorear queues.

**Instalación:**

```bash
composer require laravel/horizon

php artisan horizon:install

php artisan migrate
```

**Configurar en `config/horizon.php`:**

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['mailbox-sync'],
            'balance' => 'simple',
            'maxProcesses' => 10,
            'maxTime' => 0,
            'maxJobs' => 0,
            'memory' => 128,
            'tries' => 3,
            'timeout' => 120,
        ],
    ],
],
```

**Ejecutar Horizon:**

```bash
php artisan horizon
```

**Acceder:** http://taxo-ec.test/horizon

---

## 📊 **COMPARACIÓN: Direct vs Queue**

| Aspecto | Modo Directo (sin --queue) | Modo Queue (con --queue) |
|---------|---------------------------|--------------------------|
| **Tiempo (3025 taxpayers)** | ~51 minutos | ~5 minutos (10 workers) |
| **Paralelismo** | ❌ Secuencial | ✅ 10 procesos simultáneos |
| **Memoria** | ✅ Controlada (chunk) | ✅ Controlada (por worker) |
| **Reintentos automáticos** | ❌ No | ✅ Sí (3 intentos) |
| **Timeout** | ⚠️ Script completo | ✅ Por job individual |
| **Monitoreo** | 🟡 Progress bar CLI | ✅ Horizon + Logs |
| **Setup** | ✅ Ninguno | 🟡 Queue workers + Supervisor |
| **Escalabilidad** | ❌ Limitada | ✅ Excelente (agregar workers) |
| **Recomendado para** | <500 taxpayers | >1000 taxpayers |

---

## 🎯 **CRON SCHEDULE (Automatización)**

Para sincronizar automáticamente cada noche:

**En `app/Console/Kernel.php`:**

```php
protected function schedule(Schedule $schedule): void
{
    // Opción A: Despachar jobs (recomendado para 3025 taxpayers)
    $schedule->command('app:sync-tax-mailbox --queue')
        ->dailyAt('02:00')
        ->withoutOverlapping()
        ->runInBackground();

    // Opción B: Ejecución directa (solo si <500 taxpayers)
    // $schedule->command('app:sync-tax-mailbox')
    //     ->dailyAt('02:00')
    //     ->withoutOverlapping();
}
```

**Activar CRON:**

```bash
# Editar crontab
crontab -e

# Agregar línea:
* * * * * cd /home/isidro/code/taxo-ec && php artisan schedule:run >> /dev/null 2>&1
```

**¿Cómo funciona?**

1. **02:00 AM** → CRON ejecuta `app:sync-tax-mailbox --queue`
2. Comando despacha 3025 jobs a la cola `mailbox-sync`
3. Comando termina inmediatamente (en ~30 segundos)
4. Workers (corriendo 24/7 via Supervisor) procesan jobs en paralelo
5. Jobs terminan en ~5 minutos
6. Logs registran éxitos/fallos

---

## ⚙️ **CONFIGURACIÓN AVANZADA**

### Rate Limiting (Evitar saturar TWS API)

Si TWS tiene rate limits (ej: 100 requests/minuto):

**Crear middleware de rate limiting:**

```bash
php artisan make:middleware RateLimitTWSAPI
```

**En `app/Http/Middleware/RateLimitTWSAPI.php`:**

```php
use Illuminate\Cache\RateLimiter;

public function handle($request, Closure $next)
{
    $rateLimiter = app(RateLimiter::class);
    
    if ($rateLimiter->tooManyAttempts('tws-api', 100)) {
        // Esperar hasta que se resetee el límite
        $seconds = $rateLimiter->availableIn('tws-api');
        sleep($seconds);
    }
    
    $rateLimiter->hit('tws-api', 60); // 100 requests por 60 segundos
    
    return $next($request);
}
```

**En `SyncTaxPayerMailboxJob.php`:**

```php
use Illuminate\Queue\Middleware\RateLimited;

public function middleware()
{
    return [new RateLimited('tws-api')];
}
```

**En `app/Providers/AppServiceProvider.php`:**

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot()
{
    RateLimiter::for('tws-api', function (object $job) {
        return Limit::perMinute(100); // 100 requests por minuto
    });
}
```

---

### Job Batching (Monitorear progreso global)

Si quieres monitorear el progreso de TODOS los 3025 jobs como un lote:

**Modificar comando:**

```php
use Illuminate\Support\Facades\Bus;

private function handleWithJobs(?int $teamId, ?int $taxPayerId): int
{
    // ... código actual ...
    
    $jobs = [];
    
    $query->chunk(100, function ($taxPayers) use (&$jobs) {
        foreach ($taxPayers as $taxPayer) {
            $jobs[] = new SyncTaxPayerMailboxJob($taxPayer);
        }
    });
    
    $batch = Bus::batch($jobs)
        ->name('Sync Mailbox - ' . now()->format('Y-m-d H:i'))
        ->then(function (Batch $batch) {
            Log::info('All mailbox sync jobs completed!', [
                'total_jobs' => $batch->totalJobs,
                'processed' => $batch->processedJobs(),
                'failed' => $batch->failedJobs,
            ]);
        })
        ->catch(function (Batch $batch, \Throwable $e) {
            Log::error('Mailbox sync batch failed', [
                'error' => $e->getMessage(),
            ]);
        })
        ->finally(function (Batch $batch) {
            Log::info('Mailbox sync batch finished');
        })
        ->onQueue('mailbox-sync')
        ->dispatch();
    
    $this->info("✅ Batch creado: {$batch->id}");
    $this->info("🔍 Monitorea el progreso: http://taxo-ec.test/horizon/batches/{$batch->id}");
    
    return Command::SUCCESS;
}
```

**Ver batches:**

```bash
php artisan queue:batches

# O en Horizon:
# http://taxo-ec.test/horizon/batches
```

---

## 🚨 **TROUBLESHOOTING**

### Problema: Workers no procesan jobs

**Diagnóstico:**

```bash
# ¿Están corriendo los workers?
sudo supervisorctl status

# ¿Hay jobs en la cola?
php artisan queue:monitor mailbox-sync

# Si usas database queue:
mysql -u root -p taxo_ec -e "SELECT COUNT(*) FROM jobs WHERE queue = 'mailbox-sync';"
```

**Solución:**

```bash
# Reiniciar workers
sudo supervisorctl restart taxo-mailbox-worker:*

# Limpiar cache de configuración
php artisan config:clear
php artisan queue:restart
```

---

### Problema: Jobs fallan inmediatamente

**Diagnóstico:**

```bash
# Ver failed jobs
php artisan queue:failed

# Ver logs
tail -f storage/logs/laravel.log | grep "Failed to sync mailbox"
```

**Causas comunes:**
- ❌ Modelo `TaxPayer` no se serializa correctamente
- ❌ API de TWS está caída
- ❌ Credenciales inválidas

**Solución:**

```bash
# Reintentar failed jobs
php artisan queue:retry all

# Limpiar failed jobs muy viejos
php artisan queue:flush
```

---

### Problema: Memory exhausted en workers

**Diagnóstico:**

```bash
# Ver memoria usada por workers
ps aux | grep "queue:work" | awk '{print $6}'
```

**Solución:**

Agregar `--max-jobs` y `--max-time` en Supervisor config:

```ini
command=php /path/artisan queue:work --max-jobs=100 --max-time=3600
```

Esto reinicia workers cada 100 jobs o cada hora, liberando memoria.

---

### Problema: Jobs muy lentos

**Diagnóstico:**

```bash
# Ver jobs procesados por segundo
php artisan queue:monitor mailbox-sync
```

**Solución:**

1. **Aumentar workers:** En Supervisor config, cambiar `numprocs=10` a `numprocs=20`
2. **Optimizar query:** Revisar `TaxMailboxService::syncInbox()` para N+1 queries
3. **Usar Redis:** Cambiar de `database` a `redis` queue connection

---

## ✅ **CHECKLIST DE SETUP**

- [ ] **Configurar Queue Connection** (.env → `QUEUE_CONNECTION=database` o `redis`)
- [ ] **Crear tablas de queue** (`php artisan queue:table && php artisan migrate`)
- [ ] **Instalar Supervisor** (`sudo apt-get install supervisor`)
- [ ] **Configurar workers en Supervisor** (`/etc/supervisor/conf.d/taxo-mailbox-worker.conf`)
- [ ] **Iniciar workers** (`sudo supervisorctl start taxo-mailbox-worker:*`)
- [ ] **Probar con 1 taxpayer** (`php artisan app:sync-tax-mailbox --taxpayer=1 --queue`)
- [ ] **Verificar en logs** (`tail -f storage/logs/laravel.log`)
- [ ] **Ejecutar sync completa** (`php artisan app:sync-tax-mailbox --queue`)
- [ ] **Configurar CRON** (en `Kernel.php` + `crontab -e`)
- [ ] **[Opcional] Instalar Laravel Horizon** para UI de monitoreo

---

## 📚 **RECURSOS ADICIONALES**

- [Laravel Queues Documentation](https://laravel.com/docs/10.x/queues)
- [Laravel Horizon Documentation](https://laravel.com/docs/10.x/horizon)
- [Supervisor Documentation](http://supervisord.org/)
- [Redis Quick Start](https://redis.io/docs/getting-started/)

---

## 🎯 **RESUMEN EJECUTIVO**

**Para 3025+ taxpayers, NECESITAS usar Jobs/Queues:**

1. **Setup:** 30 minutos (una sola vez)
   ```bash
   # Database queue (más simple)
   QUEUE_CONNECTION=database
   php artisan queue:table && php artisan migrate
   
   # Supervisor (mantiene workers corriendo)
   sudo apt-get install supervisor
   # Crear /etc/supervisor/conf.d/taxo-mailbox-worker.conf
   sudo supervisorctl start taxo-mailbox-worker:*
   ```

2. **Ejecución:** 
   ```bash
   # Despacha 3025 jobs en ~30 segundos
   php artisan app:sync-tax-mailbox --queue
   
   # Workers procesan en ~5 minutos (10 workers)
   # vs ~51 minutos en modo directo
   ```

3. **Beneficios:**
   - ⚡ **10x más rápido** (5 min vs 51 min)
   - ✅ **Retry automático** (3 intentos por job)
   - ✅ **No bloquea** el servidor
   - ✅ **Escalable** (agregar más workers = más rápido)
   - ✅ **Monitoreable** (Horizon + Logs detallados)

4. **Automatización:**
   ```php
   // En Kernel.php
   $schedule->command('app:sync-tax-mailbox --queue')
       ->dailyAt('02:00');
   ```

**¡Todo listo para producción!** 🚀

