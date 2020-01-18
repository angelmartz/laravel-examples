## Response Cache Service / Middleware

- Similar to Spatie's Response Cache but will minify the html before caching. (see related service)
- Able to prime the cache via queued jobs.
- Does not serialize anything.

#### Usage: 
```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\Config;

class ResponseCache
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if(Config::get('response.cache.enabled', false)){
            return \App\Services\ResponseCache::make()->handle($next);
        }

        return $next($request);
    }
}

```

#### Service: 
```php
<?php declare(strict_types=1);

namespace App\Services;

use Illuminate\Http\Request;

use Illuminate\Support\Str;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Storage;

use Illuminate\Contracts\Filesystem\Filesystem;
use Illuminate\Contracts\Cache\Repository as CacheRepository;

use Symfony\Component\HttpFoundation\Response;

class ResponseCache
{
    protected $request;
    protected $storage;
    protected $cache;

    public function __construct(Request $request)
    {
        $this->request = $request;
        $this->storage = $this->getStorage();
        $this->cache = $this->getCache();
    }

    public static function make(): ResponseCache
    {
        return app(static::class);
    }

    public function handle(\Closure $next)
    {
        if ($this->isCached()) {
            return $this->restoreFromCache();
        }

        $response = $next($this->request);

        if ($this->shouldCache($response)) {
            $this->storeInCache($response);
        }

        return $response;
    }

    public function storeInCache(Response $response): bool
    {
        $content = $this->replaceToken($response->getContent());

        $this->storeContents($content);

        return $this->cache->forever($this->getCacheKey(), [
            'status'  => $response->getStatusCode(),
            'headers' => array_merge($response->headers->all(), [
                'Cache-Control' => 'must-revalidate,public',
                'X-Cached'      => now()->toRfc2822String(),
                'etag'          => md5($content),
            ]),
        ]);
    }

    public function restoreFromCache(): Response
    {
        $cacheKey = $this->getCacheKey();

        $data = $this->cache->get($cacheKey);

        $content = $this->restoreToken($this->readContents());

        return response($content, $data['status'], $data['headers']);
    }

    protected function shouldCache(Response $response): bool
    {
        return (
            $this->request->isMethod('GET') &&
            is_string($response->getContent()) &&
            $response->isSuccessful()
        );
    }

    protected function replaceToken(string $content): string
    {
        return str_replace(csrf_token(), config('response.cache.token_placeholder', '<TOKEN>'), $content);
    }

    protected function restoreToken(string $content): string
    {
        return str_replace(config('response.cache.token_placeholder', '<TOKEN>'), csrf_token(), $content);
    }

    protected function getCacheKey(): string
    {
        return Str::slug($this->request->path());
    }

    protected function getCacheFilePath(): string
    {
        return "response-cache/{$this->getCacheKey()}.html";
    }

    protected function readContents(): string
    {
        return $this->storage->get($this->getCacheFilePath());
    }

    protected function storeContents(string $contents): bool
    {
        return $this->storage->put($this->getCacheFilePath(), $contents);
    }

    protected function contentsExists(): bool
    {
        return $this->storage->exists($this->getCacheFilePath());
    }

    protected function isCached(): bool
    {
        return $this->cache->has($this->getCacheKey()) && $this->contentsExists();
    }

    public function flushCache(): void
    {
        $this->cache->flush();
        $this->storage->deleteDirectory('response-cache');
    }

    protected function getStorage(): Filesystem
    {
        return Storage::disk((string)config('response.cache.disk', 'local'));
    }

    protected function getCache(): CacheRepository
    {
        return Cache::store((string)config('response.cache.store', 'redis'));
    }
}
```

#### Example Job: Cache Primer 
```php
<?php declare(strict_types=1);

namespace App\Jobs;

use App\Services\ResponseCache;
use App\Services\ContentResolver;

use Illuminate\Http\Request;
use Illuminate\Support\Collection;
use Laravel\Telescope\Telescope;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Bus\Queueable;

class PrimeCacheForPages implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    /**
     * @var Collection
     */
    public $slugs;

    /**
     * Create a new job instance.
     * @param Collection $slugs
     */
    public function __construct(Collection $slugs)
    {
        $this->slugs = $slugs;
    }

    /**
     * Execute the job.
     * @return void
     */
    public function handle()
    {
        Telescope::stopRecording();

        $this->slugs->each(function($slug){

            /**
            * Make the cache service.
            * We are running in "console context" and we will need a 
            * "Faux Request" instance for the url cache key.
            */
            $cache = app(ResponseCache::class, ['request' => Request::create(url($slug))]);
            
            // Get the controller's view response programmatically.
            $response = app()->call('\App\Http\Controllers\PageController@show', ['page' => $slug]);

            // Store the response in the cache.
            $cache->storeInCache($response);
        });
    }
}
```

#### Basic Feature Tests

```php
<?php declare(strict_types=1);

namespace Tests\Unit;

use App\Jobs\PrimeCacheForPages;
use App\Services\ResponseCache;
use Illuminate\Support\Collection;
use Illuminate\Support\Facades\Config;
use Tests\TestCase;

class ResponseCacheTest extends TestCase
{
    public function test_cache_disabled()
    {
        Config::set('response.cache.enabled', false);

        ResponseCache::make()->flushCache();

        $this
            ->get('services')
            ->assertHeaderMissing('X-Cached')
            ->assertOk();

        $this
            ->get('services')
            ->assertHeaderMissing('X-Cached')
            ->assertOk();
    }

    public function test_cache_enabled()
    {
        Config::set('response.cache.enabled', true);

        ResponseCache::make()->flushCache();

        $this
            ->get(url('/services/'))
            ->assertHeaderMissing('X-Cached')
            ->assertOk();

        $this
            ->get(url('/about-us/'))
            ->assertHeaderMissing('X-Cached')
            ->assertOk();

        PrimeCacheForPages::dispatchNow('services');

        $this
            ->get(url('/services/'))
            ->assertHeader('X-Cached')
            ->assertOk();

        PrimeCacheForPages::dispatchNow('about-us');

        $this
            ->get(url('/about-us/'))
            ->assertHeader('X-Cached')
            ->assertOk();
    }

    public function test_cache_clear()
    {

        Config::set('response.cache.enabled', true);

        ResponseCache::make()->flushCache();

        PrimeCacheForPages::dispatchNow('services');

        $this
            ->get(url('/services/'))
            ->dumpHeaders()
            ->assertHeader('X-Cached')
            ->assertOk();

        ResponseCache::make()->flushCache();

        $this
            ->get(url('/services/'))
            ->assertHeaderMissing('X-Cached')
            ->assertOk();

        PrimeCacheForPages::dispatchNow('services');

        $this
            ->get(url('/services/'))
            ->assertHeader('X-Cached')
            ->assertOk();
    }
}
```