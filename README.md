


# ai-memo


では、BladeコンポーネントをAlpine.jsと連携して、タブの状態管理（アクティブなタブの切替）ができるようにします。  
Alpine.jsでは`x-data`で状態を持ち、`x-on:click`や`x-show`でUIの切り替えを行います。

## ファイル例

### 1. Tabs Bladeコンポーネント（`tabs.blade.php`）

```blade name=resources/views/components/tabs.blade.php
@props(['class' => '', 'initial' => null])

@php
    // 初期タブ値（なければ最初の子のvalueを使うのが理想ですが、シンプルにinitial属性を用意）
    $initialTab = $initial ?? 'tab1';
@endphp

<div 
    x-data="{
        activeTab: '{{ $initialTab }}'
    }"
    {{ $attributes->merge(['class' => "flex flex-col gap-2 $class", 'data-slot' => 'tabs']) }}
>
    {{ $slot }}
</div>
```

---

### 2. TabsList Bladeコンポーネント（`tabs-list.blade.php`）

```blade name=resources/views/components/tabs-list.blade.php
@props(['class' => ''])
<div {{ $attributes->merge([
    'class' => 'bg-muted text-muted-foreground inline-flex h-9 w-fit items-center justify-center rounded-lg p-[3px] ' . $class,
    'data-slot' => 'tabs-list'
]) }}>
    {{ $slot }}
</div>
```

---

### 3. TabsTrigger Bladeコンポーネント（`tabs-trigger.blade.php`）

```blade name=resources/views/components/tabs-trigger.blade.php
@props([
    'class' => '',
    'value',
    // active, disabled属性はAlpineで制御するため不要
])

<button
    type="button"
    @click="activeTab = '{{ $value }}'"
    :class="activeTab === '{{ $value }}' 
        ? 'bg-background dark:text-foreground border-input bg-input/30 shadow-sm' 
        : ''"
    :data-state="activeTab === '{{ $value }}' ? 'active' : ''"
    {{ $attributes->merge([
        'class' => 'text-foreground dark:text-muted-foreground inline-flex h-[calc(100%-1px)] flex-1 items-center justify-center gap-1.5 rounded-md border border-transparent px-2 py-1 text-sm font-medium whitespace-nowrap transition-[color,box-shadow] focus-visible:ring-[3px] focus-visible:outline-1 disabled:pointer-events-none disabled:opacity-50 data-[state=active]:shadow-sm [&_svg]:pointer-events-none [&_svg]:shrink-0 [&_svg:not([class*=\'size-\'])]:size-4 ' . $class,
        'data-slot' => 'tabs-trigger'
    ]) }}
    :disabled="{{ isset($disabled) ? 'true' : 'false' }}"
>
    {{ $slot }}
</button>
```

---

### 4. TabsContent Bladeコンポーネント（`tabs-content.blade.php`）

```blade name=resources/views/components/tabs-content.blade.php
@props(['class' => '', 'value'])

<div
    x-show="activeTab === '{{ $value }}'"
    x-cloak
    {{ $attributes->merge([
        'class' => 'flex-1 outline-none ' . $class,
        'data-slot' => 'tabs-content'
    ]) }}
>
    {{ $slot }}
</div>
```

---

## 使用例

```blade
<x-tabs initial="tab1">
    <x-tabs-list>
        <x-tabs-trigger value="tab1">タブ1</x-tabs-trigger>
        <x-tabs-trigger value="tab2">タブ2</x-tabs-trigger>
    </x-tabs-list>
    <x-tabs-content value="tab1">
        タブ1の内容
    </x-tabs-content>
    <x-tabs-content value="tab2">
        タブ2の内容
    </x-tabs-content>
</x-tabs>
```

---

### 補足
- Alpine.jsが有効な環境でご利用ください（`<script src="//unpkg.com/alpinejs" defer></script>`）。
- `initial`属性で初期アクティブタブを指定できます。
- タブ切替は`activeTab`で管理し、`x-show`で内容を切り替えます。

ご質問やカスタマイズ要望があればどうぞ！


<x-ui.tabs initial="tab1">
    <x-ui.tabs-list>
        <x-ui.tabs-trigger value="tab1">タブ1</x-ui.tabs-trigger>
        <x-ui.tabs-trigger value="tab2">タブ2</x-ui.tabs-trigger>
    </x-ui.tabs-list>
    <x-ui.tabs-content value="tab1">
        タブ1の内容
    </x-ui.tabs-content>
    <x-ui.tabs-content value="tab2">
        タブ2の内容
    </x-ui.tabs-content>
</x-ui.tabs>
