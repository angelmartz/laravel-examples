# Orderable Trait

> Contributor: https://github.com/codemonkey76

> Refactoring & Testing: https://github.com/bayareawebpro

## Schema

```
$table->bigInteger('order')->default(0);
$table->string('group')->default(null); //Any type
```

## Usage
```
class Post extends Model
{
    use Orderable;
    
    // Group by attribute with seperate order index.
    protected $orderGroup = 'group';

    protected $attributes = [
        'group'=> 'my-group' //Deafult Group
    ];

    protected $fillable = [
        'order',
        'group',
    ];

    // Required for UnitTesting with SqlLite
    protected $casts = [
        'order'=> 'int'
    ];
}

```

### Trait

```
<?php namespace App\Traits;

use Illuminate\Support\Facades\DB;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

trait Orderable
{
    //protected $orderGroup = null;

    /**
     * Increment Order & Swap
     */
    public function incrementOrder(): void
    {
        if ($this->getAttribute('isHighestOrder')) return;
        DB::transaction(function () {
            static::newQuery()
                ->scopes(['orderGroup'])
                ->where('order', $this->getAttribute('order') + 1)
                ->whereKeyNot($this->getKey())
                ->decrement('order');
            $this->increment('order');
            $this->save();
        });
    }

    /**
     * Decrement Order & Swap
     */
    public function decrementOrder(): void
    {
        if ($this->getAttribute('isLowestOrder')) return;
        DB::transaction(function () {
            static::newQuery()
                ->scopes(['orderGroup'])
                ->where('order', $this->getAttribute('order') - 1)
                ->whereKeyNot($this->getKey())
                ->increment('order');
            $this->decrement('order');
            $this->save();
        });
    }

    /**
     * Set Active Order Group
     * @param string|int $orderGroup
     */
    public function setOrderGroup($orderGroup = null): void
    {
        if(isset($orderGroup)){
            $this->orderGroup = $orderGroup;
        }
    }

    /**
     * Scope Order Group
     * @param Builder $builder
     * @param mixed $orderGroup
     */
    public function scopeOrderGroup(Builder $builder, $orderGroup = null): void
    {
        $this->setOrderGroup($orderGroup);
        if (isset($this->orderGroup)) {
            $builder->where($this->orderGroup, $this->getAttribute($this->orderGroup));
        }
    }

    /**
     * Scope Higher Than
     * @param Builder $builder
     * @param int $order
     * @param null $group
     */
    public function scopeHigherThan(Builder $builder, int $order, $group = null): void
    {
        $builder
            ->scopes(['orderGroup'])
            ->where('order', '>', $order)
            ->orderBy('order', 'ASC');
    }

    /**
     * Scope Lower Than
     * @param Builder $builder
     * @param int $order
     * @return void
     */
    public function scopeLowerThan(Builder $builder, int $order): void
    {
        $builder
            ->scopes(['orderGroup'])
            ->where('order', '<', $order)
            ->orderBy('order', 'DESC');
    }

    /**
     * Get Highest Order Attribute
     * @return int
     */
    public function getHighestOrderAttribute(): int
    {
        return (int) static::newQuery()->scopes(['orderGroup'])->max('order');
    }

    /**
     * Get Lowest Order Attribute
     * @return int
     */
    public function getLowestOrderAttribute(): int
    {
        return (int) static::newQuery()->scopes(['orderGroup'])->min('order');
    }

    /**
     * Is Self Lowest?
     * @return bool
     */
    public function getIsLowestOrderAttribute(): bool
    {
        return (bool)($this->getAttribute('lowestOrder') === $this->getAttribute('order'));
    }

    /**
     * Is Self Highest?
     * @return bool
     */
    public function getIsHighestOrderAttribute(): bool
    {
        return (bool)($this->getAttribute('highestOrder') === $this->getAttribute('order'));
    }

    /**
     * Boot Orderable Trait
     * @return void
     */
    public static function bootOrderable()
    {
        static::saving(function (Model $model) {

            if(empty($model->getAttribute('order'))){
                $model->setAttribute('order', $model->getAttribute('highestOrder') + 1);
            }
        });
    }
}

```

## Unit Test
```
<?php
namespace Tests\Unit;

use App\Post;
use Illuminate\Support\Collection;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class OrderableTest extends TestCase
{
    use RefreshDatabase;

    /**
     * A basic test example.
     * @return void
     */
    public function testBasicTest()
    {
        Post::query()->delete();
        /**
         * @var $postA Post
         * @var $postB Post
         * @var $postC Post
         * @var $postD Post
         */
        $postA = tap(Post::create(['group' => 'base']))->save();
        $postB = tap(Post::create(['group' => 'base']))->save();
        $postC = tap(Post::create(['group' => 'base']))->save();
        $postD = tap(Post::create(['group' => 'base']))->save();
        $postE = tap(Post::create(['group' => 'alternate']))->save();

        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $postE->refresh();

        dump(Collection::make([
            "A" => $postA->toArray(),
            "B" => $postB->toArray(),
            "C" => $postC->toArray(),
            "D" => $postD->toArray(),
            "E" => $postE->toArray(),
        ])->sort());

        $this->assertTrue($postA->lowestOrder === 1, 'A => 1 lowestOrder');
        $this->assertTrue($postA->highestOrder === 4, 'A => 4  highestOrder');
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'C 3');
        $this->assertTrue($postA->isLowestOrder, 'A self lowest');
        $this->assertTrue($postD->isHighestOrder, 'D self highest');

        $this->assertTrue($postE->order === 1, 'E 1');
        $this->assertTrue($postE->group === 'alternate', 'E group');
        $this->assertTrue($postE->isLowestOrder, 'A self lowest');
        $this->assertTrue($postE->isHighestOrder, 'D self highest');

        //Ignore Decrement of Lowest
        $postA->decrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');

        //Increment Lowest Value
        $postA->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 2, 'A 2');
        $this->assertTrue($postB->order === 1, 'B 1');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');

        //Increment New Lowest Value
        $postB->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 3, 'C 3');
        $this->assertTrue($postD->order === 4, 'D 4');

        //Increment 2nd Highest Value
        $postC->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 1, 'A 1');
        $this->assertTrue($postB->order === 2, 'B 2');
        $this->assertTrue($postC->order === 4, 'C 4');
        $this->assertTrue($postD->order === 3, 'D 3');

        //Increment 2nd Highest Value
        $postD->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 1, 'A');
        $this->assertTrue($postB->order === 2, 'B');
        $this->assertTrue($postC->order === 3, 'C');
        $this->assertTrue($postD->order === 4, 'D');

        //Ignore Increment of Highest
        $postD->incrementOrder();
        $postA->refresh();
        $postB->refresh();
        $postC->refresh();
        $postD->refresh();
        $this->assertTrue($postA->order === 1, 'A');
        $this->assertTrue($postB->order === 2, 'B');
        $this->assertTrue($postC->order === 3, 'C');
        $this->assertTrue($postD->order === 4, 'D');

        dump(Collection::make([
            "A" => $postA->order,
            "B" => $postB->order,
            "C" => $postC->order,
            "D" => $postD->order,
        ])->sort());
    }
}
```