了解です！  
**APIではなくWeb（Blade等の画面）で動作する**、  
**ビジネスカードの承認制トレード機能**の  
**ルーティングからController・モデル・Bladeまでの実装例**を  
**step by stepで具体的に解説**します。

---

## 1. ルーティングの設定（`routes/web.php`）

```php name=routes/web.php
use App\Http\Controllers\BusinessCardTradeController;

// 申請フォーム（カード一覧等から申請できる）
Route::post('/trades/request', [BusinessCardTradeController::class, 'tradeRequest'])->name('trades.request');

// 申請一覧（自分が送った・受け取った申請を見る）
Route::get('/trades', [BusinessCardTradeController::class, 'index'])->name('trades.index');

// 承認・拒否（相手が申請を承認/拒否する）
Route::post('/trades/{trade}/accept', [BusinessCardTradeController::class, 'accept'])->name('trades.accept');
Route::post('/trades/{trade}/reject', [BusinessCardTradeController::class, 'reject'])->name('trades.reject');
```

---

## 2. モデルの用意（`BusinessCardTrade.php`）

```php name=app/Models/BusinessCardTrade.php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class BusinessCardTrade extends Model
{
    use HasFactory;

    protected $fillable = [
        'sender_id', 'receiver_id', 'card_id', 'status'
    ];

    // 送信者
    public function sender()
    {
        return $this->belongsTo(User::class, 'sender_id');
    }

    // 受信者
    public function receiver()
    {
        return $this->belongsTo(User::class, 'receiver_id');
    }

    // ビジネスカード
    public function card()
    {
        return $this->belongsTo(BusinessCard::class, 'card_id');
    }
}
```

---

## 3. コントローラー実装例（`BusinessCardTradeController.php`）

```php name=app/Http/Controllers/BusinessCardTradeController.php
<?php

namespace App\Http\Controllers;

use App\Models\BusinessCard;
use App\Models\BusinessCardTrade;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class BusinessCardTradeController extends Controller
{
    // 申請一覧（送信・受信両方）
    public function index()
    {
        $user = Auth::user();
        $sentTrades = BusinessCardTrade::where('sender_id', $user->id)->with('receiver', 'card')->latest()->get();
        $receivedTrades = BusinessCardTrade::where('receiver_id', $user->id)->with('sender', 'card')->latest()->get();
        return view('trades.index', compact('sentTrades', 'receivedTrades'));
    }

    // トレード申請
    public function tradeRequest(Request $request)
    {
        $request->validate([
            'card_id' => 'required|exists:business_cards,id',
            'receiver_id' => 'required|exists:users,id|different:' . Auth::id(),
        ]);

        // 自分のカードのみ申請可能
        $card = BusinessCard::where('id', $request->card_id)->where('user_id', Auth::id())->firstOrFail();

        // 重複申請防止
        $exists = BusinessCardTrade::where('sender_id', Auth::id())
            ->where('receiver_id', $request->receiver_id)
            ->where('card_id', $card->id)
            ->where('status', 'pending')
            ->exists();

        if ($exists) {
            return back()->with('error', 'すでに申請中です。');
        }

        BusinessCardTrade::create([
            'sender_id' => Auth::id(),
            'receiver_id' => $request->receiver_id,
            'card_id' => $card->id,
            'status' => 'pending',
        ]);

        return back()->with('success', '申請しました！');
    }

    // 承認
    public function accept($id)
    {
        $trade = BusinessCardTrade::where('id', $id)
            ->where('receiver_id', Auth::id())
            ->where('status', 'pending')
            ->firstOrFail();

        $trade->status = 'accepted';
        $trade->save();

        // 必要ならここで通知やカードの複製など処理

        return back()->with('success', '交換を承認しました！');
    }

    // 拒否
    public function reject($id)
    {
        $trade = BusinessCardTrade::where('id', $id)
            ->where('receiver_id', Auth::id())
            ->where('status', 'pending')
            ->firstOrFail();

        $trade->status = 'rejected';
        $trade->save();

        // 必要ならここで通知など処理

        return back()->with('info', '申請を拒否しました。');
    }
}
```

---

## 4. Bladeテンプレート例

### 申請ボタン（カード一覧等に配置）

```blade name=resources/views/business_cards/show.blade.php
{{-- ...カード表示部分... --}}
@if(Auth::id() != $user->id)
<form action="{{ route('trades.request') }}" method="POST">
    @csrf
    <input type="hidden" name="card_id" value="{{ $card->id }}">
    <input type="hidden" name="receiver_id" value="{{ $user->id }}">
    <button type="submit" class="btn btn-primary">このカードを交換申請</button>
</form>
@endif
```

---

### 申請一覧・承認/拒否画面

```blade name=resources/views/trades/index.blade.php
@extends('layouts.app')

@section('content')
<h2>受け取った申請</h2>
@foreach($receivedTrades as $trade)
    <div>
        {{ $trade->sender->name }} さんから
        カード「{{ $trade->card->name }}」の交換申請
        [状態: {{ $trade->status }}]
        @if($trade->status === 'pending')
            <form action="{{ route('trades.accept', $trade) }}" method="POST" style="display:inline">
                @csrf
                <button type="submit">承認</button>
            </form>
            <form action="{{ route('trades.reject', $trade) }}" method="POST" style="display:inline">
                @csrf
                <button type="submit">拒否</button>
            </form>
        @endif
    </div>
@endforeach

<h2>送った申請</h2>
@foreach($sentTrades as $trade)
    <div>
        {{ $trade->receiver->name }} さんに申請した
        カード「{{ $trade->card->name }}」
        [状態: {{ $trade->status }}]
    </div>
@endforeach
@endsection
```

---

## 5. 補足

- **認証ミドルウェア**：ルートに`middleware('auth')`をつけてください。
- **バリデーション・エラーメッセージ**：ブラウザで表示する仕組みは適宜整えます。
- **通知・カード複製など追加処理**：必要なら`accept`内で実装。

---

## まとめ

1. **web.phpでルート定義**
2. **モデルにリレーション**
3. **コントローラで申請・承認・拒否のロジック**
4. **Bladeで申請ボタン/申請一覧画面/承認・拒否ボタンのUI**

この流れで、  
**トレード申請→相手が承認/拒否→履歴の管理**  
までWeb画面で実装できます！

さらに細かい部分やカスタマイズもご相談ください！
