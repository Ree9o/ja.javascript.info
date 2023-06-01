libs:
  - d3
  - domtree

---

# 選択（Selection） と 範囲（Range）

この章ではドキュメントでの選択と、`<input>` などのフォームフィールドでの選択について説明します。

JavaScript を利用して選択状態を取得したり、全体あるいは一部分の選択/選択解除、ドキュメントから選択した部分を削除、タグへのラップなどを行うことができます。

末尾の "サマリ" セクションにレシピがあり、これで現時点で必要なことはカバーされているかもしれません。ただ、章全体を読むことでより多くのことを知ることができます。

基礎となる `Range` と `Selection` オブジェクトの把握は難しくはないので、必要なことをするためのレシピは必要ありません。

## 範囲(Range)

選択の基本的な概念は [範囲(Range)](https://dom.spec.whatwg.org/#ranges) で、基本的には "境界点"(範囲の開始と終了) のペアです。

`Range` オブジェクトはパラメータなしで作成できます:

```js
let range = new Range();
```

次に、`range.setStart(node, offset)` と `range.setEnd(node, offset)` を使用して選択の境界を設定します。

ご想像のとおり、さらに選択をしていくために `Range` オブジェクトを使用しますが、最初にそのようなオブジェクトをいくつか作成しましょう。

### テキストを部分的に選択

興味深いことは、両方のメソッドの最初の引数 `node` はテキストノード、あるいは要素ノードで、2つ目の引数の意味はその種類によります。

**`node` がテキストノードの場合、`offset` はそのテキストでの位置になります。**

例えば、要素 `<p>Hello</p>` がある場合、次のようにして文字 "ll" を含む範囲を作成できます。: 

```html run
<p id="p">Hello</p>
<script>
  let range = new Range();
  range.setStart(p.firstChild, 2);
  range.setEnd(p.firstChild, 4);
  
  // range の toString はテキストとしてコンテンツを返します
  console.log(range); // ll
</script>
```

ここでは `<p>` の最初の子（テキストノード）を取り、その中のテキスト位置を指定しています。:

![](range-hello-1.svg)

### 要素ノードの選択

**一方、`node` が要素ノードであれば、`offset` は子供の番号になります。**

これは、テキスト内のどこかで停止するのではなく、ノード全体を含む範囲を作成する場合に便利です。

例として、この HTML の一部を考えます:

```html
<p id="p">Example: <i>italic</i> and <b>bold</b></p>
```

DOM構造は次の通りです。ここではテキストノードが重要です。:

<div class="select-p-domtree"></div>

<script>
let selectPDomtree = {
  "name": "P",
  "nodeType": 1,
  "children": [{
    "name": "#text",
    "nodeType": 3,
    "content": "Example: "
  }, {
    "name": "I",
    "nodeType": 1,
    "children": [{
      "name": "#text",
      "nodeType": 3,
      "content": "italic"
    }]
  }, {
    "name": "#text",
    "nodeType": 3,
    "content": " and "
  }, {
    "name": "B",
    "nodeType": 1,
    "children": [{
      "name": "#text",
      "nodeType": 3,
      "content": "bold"
    }]
  }]
}

drawHtmlTree(selectPDomtree, 'div.select-p-domtree', 690, 320);
</script>

`"Example: <i>italic</i>"` の範囲を作成しましょう。

ご覧の通り、これはインデックス `0` と `1` を持つ `<p>` の先頭から2つの子です:

![](range-example-p-0-1.svg)

- 開始位置は親の `node` として `<p>` を、オフセットは `0` を持ちます。 
    
    なので、`range.setStart(p, 0)` と設定できます。
- 終了位置も親ノードとして `<p>` を持ちますが、オフセットとしては `2` になります（ここまで、という範囲を指定しますが、`offset` の含みません）。

    そのため、 `range.setEnd(p, 2)` と設定できます。

これはデモです。実行するとテキストが選択されるのが分かります:

```html run
<p id="p">Example: <i>italic</i> and <b>bold</b></p>

<script>
*!*
  let range = new Range();

  range.setStart(p, 0);
  range.setEnd(p, 2);
*/!*

  // range の toString はそのコンテンツをテキストとして(タグなし)返します
  alert(range); // Example: italic

  // この range をドキュメント選択に適用します（後で説明します）
  document.getSelection().addRange(range);
</script>
```

これはより柔軟な例で多くのパターンを試せます。:

```html run autorun
<p id="p">Example: <i>italic</i> and <b>bold</b></p>

From <input id="start" type="number" value=1> – To <input id="end" type="number" value=4>
<button id="button">Click to select</button>
<script>
  button.onclick = () => {
  *!*
    let range = new Range();

    range.setStart(p, start.value);
    range.setEnd(p, end.value);
  */!*

    // apply the selection, explained later
    document.getSelection().removeAllRanges();
    document.getSelection().addRange(range);
  };
</script>
```

例. `1` から `4` を選択した場合の範囲は `<i>italic</i> and <b>bold</b>` です。

![](range-example-p-1-3.svg)

```smart header="開始ノードと終了ノードは異なっていても問題ありません"
`setStart` と `setEnd` で同じノードを指定する必要はありません。範囲（range）は多数の無関係なノードにまたがる場合があります。文書内で終了が開始の後にあることだけが重要です。
```

### より大きなフラグメントの選択

次のように、例の中でより大きな選択を作成してみましょう:

![](range-example-p-2-b-3.svg)

すでにやり方は知っています。開始と終了をテキストノードで相対オフセットとしてセットするだけです。

次のような範囲を作成します:
- `<p>` の最初の子の位置 2 から開始("Ex<b>ample:</b> " の最初の2文字を除くすべて)
- `<b>` の最初の子の位置 3 で終了("<b>bol</b>d" の最初の3文字):

```html run
<p id="p">Example: <i>italic</i> and <b>bold</b></p>

<script>
  let range = new Range();

  range.setStart(p.firstChild, 2);
  range.setEnd(p.querySelector('b').firstChild, 3);

  alert(range); // ample: italic and bol

  // 選択にこの範囲を使用します(後ほど説明します)
  window.getSelection().addRange(range);
</script>
```

ご覧の通り、必要とする範囲（range）を作成するのは非常に簡単です。

もしノード全体をトリたい場合、`setStart/setEnd` で要素を渡すことができます。それ以外の場合はテキストレベルで作業できます。

## 範囲（range）プロパティ

上の例で作成した範囲（range）オブジェクトは次のプロパティを持ちます:

![](range-example-p-2-b-3-range.svg)

- `startContainer`, `startOffset` -- 開始点のノードとオフセット
  - 上の例では、`p` 内の最初のテキストノートと `2` です。
- `endContainer`, `endOffset` -- 終了点のノードとオフセット
  - 上の例では、`<b>` 内の最初のテキストノードと `3` です。
- `collapsed` -- 真偽値, range の開始/終了点が同じ(つまり range 内にコンテンツがない)場合は `true` です。
  - 上の例では `false`　です。
- `commonAncestorContainer` -- range 内のすべてのノードの最も近い共通の祖先
  - 上の例では `<p>` です。


## 範囲(range) 選択メソッド

範囲を操作するための便利なメソッドがたくさんあります。

すでに `setStart` と `setEnd` は見てきましたが、ここには他の類似メソッドがあります。

範囲の開始を設定:

- `setStart(node, offset)` は `node` 内の `offset` の位置に開始点を設定します。
- `setStartBefore(node)` は `node` の直前を開始点に設定します。
- `setStartAfter(node)` は `node` の直後を開始点に設定します。

範囲の終了を設定(同様のメソッドです):

- `setEnd(node, offset)` は `node` 内の `offset` の位置に終了点を設定します。
- `setEndBefore(node)`  `node` の直前を終了点に設定します。
- `setEndAfter(node)` は `node` の直後を終了点に設定します。

技術的には、`setStart/setEnd` は何でもできますが、より多くのメソッドでさらに便利になります。

これらのメソッドはすべて、`node` はテキストあるいは要素ノード両方になれます。テキストノードの場合 `offset` は文字をスキップする一方で、要素ノードは子ノードをスキップします。

範囲（range）を作成するためのメソッドが他にもあります:
- `selectNode(node)` は `node` 全体を選択するような範囲を設定します。
- `selectNodeContents(node)` は `node` のコンテンツ全体を選択するような範囲を設定します
- `collapse(toStart)` は、`toStart=true` の場合 end=start 、そうでなければ start=end を設定します。範囲を折りたたみます。
- `cloneRange()` は同じ開始/終了点をもつ新しい範囲を作成します。

## 範囲（range）編集メソッド

一度範囲(range)を作成し、次のようなメソッドを使用することでコンテンツを操作することができます:

- `deleteContents()` - ドキュメントから範囲のコンテンツを削除します
- `extractContents()` - ドキュメントから範囲のコンテンツを削除し、[DocumentFragment](info:modifying-document#document-fragment) として返却します。
- `cloneContents()` - 範囲のコンテンツをクローンし、[DocumentFragment](info:modifying-document#document-fragment) として返却します。
- `insertNode(node)` -- ドキュメントの範囲の先頭に `node` を挿入します。
- `surroundContents(node)` -- `node` で範囲コンテンツをラップします。これが機能するには、範囲内にすべての要素の開始と終了タグが含まれている必要があります。`<i>abc` のような部分的な範囲では機能しません。

これらのメソッドを使用すると、選択したノードに対し基本的に何でもできます。

これは実際の動作が確認できる例です。:

```html run autorun height=260
ボタンクリックで選択範囲に対しメソッドを実行し、"resetExample" でリセットします。

<p id="p">Example: <i>italic</i> and <b>bold</b></p>

<p id="result"></p>
<script>
  let range = new Range();

  // Each demonstrated method is represented here:
  let methods = {
    deleteContents() {
      range.deleteContents()
    },
    extractContents() {
      let content = range.extractContents();
      result.innerHTML = "";
      result.append("extracted: ", content);
    },
    cloneContents() {
      let content = range.cloneContents();
      result.innerHTML = "";
      result.append("cloned: ", content);
    },
    insertNode() {
      let newNode = document.createElement('u');
      newNode.innerHTML = "NEW NODE";
      range.insertNode(newNode);
    },
    surroundContents() {
      let newNode = document.createElement('u');
      try {
        range.surroundContents(newNode);
      } catch(e) { alert(e) }
    },
    resetExample() {
      p.innerHTML = `Example: <i>italic</i> and <b>bold</b>`;
      result.innerHTML = "";

      range.setStart(p.firstChild, 2);
      range.setEnd(p.querySelector('b').firstChild, 3);

      window.getSelection().removeAllRanges();  
      window.getSelection().addRange(range);  
    }
  };

  for(let method in methods) {
    document.write(`<div><button onclick="methods.${method}()">${method}</button></div>`);
  }

  methods.resetExample();
</script>
```

範囲を比較するメソッドも存在しますが、めったに使われることはありません。必要になったら、[仕様](https://dom.spec.whatwg.org/#interface-range) や [MDN マニュアル](https://developer.mozilla.org/en-US/docs/Web/API/Range)を参照してください。


## 選択(Selection)

`Range` は選択範囲を管理するための汎用オブジェクトです。このようなオブジェクトを作成し利用しますが-- それらは自身では視覚的には何も選択しません。

ドキュメントの選択は `Selection` オブジェクトで表現され、`window.getSelection()` あるいは `document.getSelection()` で取得することができます。

選択は 0 個以上の範囲を含めることができます。すくなくとも [Selection API 仕様](https://www.w3.org/TR/selection-api/) ではそのように述べています。ですが、実際には Firefox のみが `key:Ctrl+click` (Mac の場合は `key:Cmd+click`)を使用してドキュメント内の複数の範囲を選択することができます。

これは Firefox で 3つの範囲を選択しているスクリーンショットです。:

![](selection-firefox.svg)

他のブラウザは最大1つの範囲だけサポートしています。後で見ていきますが、`Selection` メソッドによっては複数の範囲が存在する可能性があることを示唆しているものもあります。が、繰り返しになりますが Firefox 以外のブラウザは最大で 1 です。

これは小さなデモで、現在の選択（何かを選択してクリック）をテキストとして表示します。

<button onclick="alert(document.getSelection())">alert(document.getSelection())</button>

## Selection プロパティ

前述したように、理論上、選択には複数の範囲が含まれる場合があります。これらの範囲オブジェクトは、次のメソッドを使用して取得できます。

- `getRangeAt(i)` -- `0` から始まる i 番目の範囲を取得します。Firefox を除くすべてのブラウザでは、`0` のみが使用されます。

また、多くの場合、利便性が向上するプロパティも存在します。

範囲(range)と同様、選択には始点と終点があり、それぞれ "anchor(アンカー)"、"focus(フォーカス)" と呼ばれます。

主な selection プロパティは次のものです:

- `anchorNode` -- selection の始点のある node です。
- `anchorOffset` -- selection の始点の `anchorNode` でのオフセットです。
- `focusNode` -- selection の終点のある node です。
- `focusOffset` -- selection の終点の `focusNode` でのオフセットです。
- `isCollapsed` -- selection が未選択(空の範囲) あるいは存在しない場合 `true` になります。
- `rangeCount` -- selection に含まれる range の数です。

````smart header="選択(selection) 終了/開始 vs 範囲(range)"

選択(selection)の アンカー/フォーカス は `Range` の 開始/終了 と比較して重要な違いがあります。

ご存知の通り、`Range` オブジェクトは常に 終了の前に開始があります。

選択の場合、常にそうであるわけではありません。

マウスで何かを選択することは、「左から右」または「右から左」の両方向で行うことができます。

つまり、マウス ボタンを押してドキュメント内を前に移動すると、その終了点 (フォーカス) は開始点 (アンカー) の後に来ます。

E.g. ユーザがマウスで選択を開始し、"Example" から "italic" まで操作した場合:

![](selection-direction-forward.svg)

...しかし、同じ選択を逆方向に行うこともできます。"italic" から "Example" まで開始すると (逆方向)、その終了 (フォーカス) が開始 (アンカー) の前になります。

![](selection-direction-backward.svg)
````

## Selection イベント

選択範囲を追跡するためのイベントがあります:

- `elem.onselectstart` -- `elem`(またはその内部)で選択が *開始* されたとき。例えば、ユーザがボタンを押しながらマウスを動かし始めたとき。
    - デフォルトアクションを防いだ場合、選択は開始されません。したがって、この要素から選択を開始することは不可能になりますが、要素は引き続き選択可能です。訪問者は他の場所から選択を開始するだけで済みます。
- `document.onselectionchange` -- 選択範囲が変更されたとき。
    - 注意: このハンドラは `document` に対してのみ設定可能です。

### 選択範囲の追跡デモ

これは選択境界の変更に応じて動的に選択境界を表示する小さなデモです:

```html run height=80
<p id="p">Select me: <i>italic</i> and <b>bold</b></p>

From <input id="from" disabled> – To <input id="to" disabled>
<script>
  document.onselectionchange = function() {
    let selection = document.getSelection();

    let {anchorNode, anchorOffset, focusNode, focusOffset} = selection;

    // anchorNode と focusNode は通常テキストノードです
    from.value = `${anchorNode?.data}, offset ${anchorOffset}`;
    to.value = `${focusNode?.data}, offset ${focusOffset}`;
  };
</script>
```

### 選択(selection)のコピーデモ

選択されたコンテンツをコピーする、2つのアプローチがあります:

1. `document.getSelection().toString()` を使用してテキストとして取得できます。
2. DOM ノードとして: 基底となる範囲を取得し、それらの `cloneContents()` を呼び出します(Firefox のマルチ選択をサポートしてない場合は最初の1つの range に対してのみ)。

そして、これはテキストとDOM ノード両方で選択範囲を取得するデモです:

```html run height=100
<p id="p">Select me: <i>italic</i> and <b>bold</b></p>

Cloned: <span id="cloned"></span>
<br>
As text: <span id="astext"></span>

<script>
  document.onselectionchange = function() {
    let selection = document.getSelection();

    cloned.innerHTML = astext.innerHTML = "";

    // range から DOM ノードをクローンします(ここでは multiselect をサポートしています)
    for (let i = 0; i < selection.rangeCount; i++) {
      cloned.append(selection.getRangeAt(i).cloneContents());
    }

    // テキストとして取得
    astext.innerHTML += selection;
  };
</script>
```

## Selection メソッド

range の追加/削除をするための Selection メソッド:

- `getRangeAt(i)` -- `0` から始まる i 番目の range を取得します。firefox 以外のブラウザは `0` だけが使用されます。
- `addRange(range)` -- 選択範囲に `range` を追加します。すでに range が関連付けられている場合、firefox 以外のブラウザは呼び出しを無視します。
- `removeRange(range)` -- selection から `range` を削除します。
- `removeAllRanges()` -- すべての range を削除します。
- `empty()` -- `removeAllRanges` のエイリアスです。

また、`Range` なしで選択範囲をを直接操作するための便利なメソッドがあります:

- `collapse(node, offset)` -- 選択された range を、指定された `node` の位置 `offset` で開始及び終了する新しい range に置き換えます。
- `setPosition(node, offset)` -- `collapse` のエイリアスです。
- `collapseToStart()` - 選択範囲の始点に折りたたみます(空の range に置き換えます)
- `collapseToEnd()` - 選択範囲の終点に折りたたみます
- `extend(node, offset)` - 選択範囲の focus を指定された `node` の位置 `offset` に移動します。 
- `setBaseAndExtent(anchorNode, anchorOffset, focusNode, focusOffset)` - 選択範囲の range を、指定された始点 `anchorNode/anchorOffset` と終点 `focusNode/focusOffset` に置き換えます。これらの間にあるすべてのコンテンツが選択されます。
- `selectAllChildren(node)` -- `node` のすべての子を選択します。
- `deleteFromDocument()` -- ドキュメントから選択されたコンテンツを削除します。
- `containsNode(node, allowPartialContainment = false)` -- 選択範囲が `node` を含むかチェックします(2番めの引数が `true` の場合は部分的に含む、を許可する)。

したがって、多くのタスクで `Selection` メソッドを呼び出すことができ、基礎となる `Range` オブジェクトにアクセスする必要はありません。 

例えば、段落 `<p>` のコンテンツ全体を選択するには次のようにします:

```html run
<p id="p">Select me: <i>italic</i> and <b>bold</b></p>

<script>
  // <p> の 0 番目の子から最後の子までを選択
  document.getSelection().setBaseAndExtent(p, 0, p, p.childNodes.length);
</script>
```

The same thing using ranges:

```html run
<p id="p">Select me: <i>italic</i> and <b>bold</b></p>

<script>
  let range = new Range();
  range.selectNodeContents(p); // or selectNode(p) で <p> タグも選択します

  document.getSelection().removeAllRanges(); // 存在する選択範囲をクリアします
  document.getSelection().addRange(range);
</script>
```

```smart header="選択するには、最初に既存の選択範囲を削除してください"
選択範囲がすでに存在する場合、`removeAllRanges()` で最初に空にし、その後 range を追加してください。そうでない場合、Firefox 以外のブラウザは新しい range を無視します。

`setBaseAndExtent` などのいくつかの selection メソッドは例外で、既存の選択範囲を置き換えます。
```

## フォームコントロールでの選択

`input` や `textarea` と行ったフォーム要素は `Selection` や `Range` オブジェクトなしで、[選択のための特別なAPI](https://html.spec.whatwg.org/#textFieldSelection) を提供します。入力値は HTML ではなく純粋なテキストなので、このようなオブジェクトは必要なく、すべてがよりシンプルです。

プロパティ:
- `input.selectionStart` -- 選択開始位置（書き込み可能）
- `input.selectionEnd` -- 選択終了位置（書き込み可能）
- `input.selectionDirection` -- 選択方向。次のいずれかです: "forward", "backward" あるいは "none" (例 マウスのダブルクリックで選択した場合 ) です。

イベント:
- `input.onselect` -- 何かが選択されたときに発生します。

メソッド:

- `input.select()` -- テキストコントロール内のすべてを選択します（`input` の代わりに `textarea` も OK です）。
- `input.setSelectionRange(start, end, [direction])` -- 選択範囲を、`start` 位置から `end` 位置まで、指定された方向(任意)に広げます。
- `input.setRangeText(replacement, [start], [end], [selectionMode])` -- テキストの範囲を新しいテキストに置き換えます。

    任意の引数 `start` と `end` が指定された場合は開始/終了の範囲を設定します。そうでない場合はユーザの選択を使用されます。

    最後の引数 `selectionMode` はテキストが置換されたあとの選択方法を決定します。次の値が指定できます: 

    - `"select"` -- 新たに挿入されたテキストが選択されます。
    - `"start"` -- 選択範囲は挿入されたテキストの直前に折りたたまれます(カーソルはその直前になります)。
    - `"end"` -- 選択範囲は挿入されたテキストの直後に折りたたまれます(カーソルはその直後になります)。
    - `"preserve"` -- 選択範囲を保とうとします。デフォルトはこの動作になります。

それでは、これらのメソッドの動作を見てみましょう。

### 例: 選択の追跡

例えば、このコードは `onselect` イベントを使用して選択を追跡します:

```html run autorun
<textarea id="area" style="width:80%;height:60px">
Selecting in this text updates values below.
</textarea>
<br>
From <input id="from" disabled> – To <input id="to" disabled>

<script>
  area.onselect = function() {
    from.value = area.selectionStart;
    to.value = area.selectionEnd;
  };
</script>
```

注意してください:
- `onselect` はなにかが選択されたときに発生しますが、選択が除去されたときには発生しません。
- `document.onselectionchange` イベントは、[仕様](https://w3c.github.io/selection-api/#dfn-selectionchange)によると `document` の選択や範囲とは関係ないため、フォームコントロール内の選択では発生すべきではありません。一部のブラウザにはイベントを生成しますが、それに頼るべきではありません。


### 例: カーソルの移動

選択範囲を設定する `selectionStart` と `selectionEnd` を変更することができます。

重要なエッジケースは `selectionStart` と `selectionEnd` が互いに等しい場合です。そのとき、それはまさにカーソル位置になります。あるいは何も選択されていない場合は、選択はカーソル位置で折りたたまれます。

したがって、`selectionStart` と `selectionEnd` を同じ値に設定することでカーソルを移動させることができます。

例:

```html run autorun
<textarea id="area" style="width:80%;height:60px">
Focus on me, the cursor will be at position 10.
</textarea>

<script>
  area.onfocus = () => {
    // ブラウザの "focus" アクションが終了したあとに実行させるためのゼロ遅延の setTimeout
    setTimeout(() => {
      // 任意の選択を設定できます
      // 開始 = 終了 の場合、カーソルはまさにその場所です
      area.selectionStart = area.selectionEnd = 10;
    });
  };
</script>
```

### 例: 選択の変更

選択内容を変更するには、`input.setRangeText()` メソッドが利用できます。もちろん、`selectionStart/End` を読み取り、選択に関する知識があれば `value` の対応する部分文字列を変更することはできますが、`setRangeText` はより強力で、多くの場合より便利です。

これはいくらか複雑なメソッドです。最もシンプルな単一引数のフォームでは、ユーザの選択した範囲を置き換え、選択を削除します。

例えば、ここではユーザの選択は `*...*` で囲まれます:

```html run autorun
<input id="input" style="width:200px" value="Select here and click the button">
<button id="button">Wrap selection in stars *...*</button>

<script>
button.onclick = () => {
  if (input.selectionStart == input.selectionEnd) {
    return; // 何も選択されていない
  }

  let selected = input.value.slice(input.selectionStart, input.selectionEnd);
  input.setRangeText(`*${selected}*`);
};
</script>
```

引数を増やすことで、範囲の `start` と `end` を設定することができます。

この例では、入力テキストから `"THIS"` を見つけ、それを置き換え、置換された選択範囲を維持します:

```html run autorun
<input id="input" style="width:200px" value="Replace THIS in text">
<button id="button">Replace THIS</button>

<script>
button.onclick = () => {
  let pos = input.value.indexOf("THIS");
  if (pos >= 0) {
    input.setRangeText("*THIS*", pos, pos + 4, "select");
    input.focus(); // 選択を見えるようにするための focus
  }
};
</script>
```

### 例: カーソル位置に挿入

何も選択されていない場合、あるいは `setRangeText` で `start` と `end` を等しくした場合、新しいテキストが単に挿入されるだけで何も削除はされません。

`setRangeText` を使用して "カーソル位置" に挿入することもできます。

これはカーソル位置に `"HELLO"` を挿入し、その直後にカーソルを置くボタンです。選択が空でない場合は置き換えられます(`selectionStart!=selectionEnd` を比較することでこれを検知し、代わりになにかを行うことができます):

```html run autorun
<input id="input" style="width:200px" value="Text Text Text Text Text">
<button id="button">Insert "HELLO" at cursor</button>

<script>
  button.onclick = () => {
    input.setRangeText("HELLO", input.selectionStart, input.selectionEnd, "end");
    input.focus();
  };    
</script>
```


## 選択不可にする

選択不可にするための３つの方法があります:

1. CSS プロパティ `user-select: none` を使用します.

    ```html run
    <style>
    #elem {
      user-select: none;
    }
    </style>
    <div>Selectable <div id="elem">Unselectable</div> Selectable</div>
    ```

    これは `elem` から始まる選択を許可しません。ですが、ユーザは別の場所から選択を開始して、その中に `elem` を含むかもしれません。

    すると、`elem` は`document.getSelection()` の一部になるので、選択は実際には起こります。が、その中身は通常コピー/貼り付けでは無視されます。


2. `onselectstart` または `mousedown` イベントで、デフォルト動作を防止します。

    ```html run
    <div>Selectable <div id="elem">Unselectable</div> Selectable</div>

    <script>
      elem.onselectstart = () => false;
    </script>
    ```

    これにより `elem` から選択を開始するのことはできなくなりますが、訪問者は別の要素で選択を開始して `elem` まで広げることができます。

    これは、選択をトリガーする同じアクションに別のイベントハンドラーがある場合に便利です(例. `mousedown`)。そのため、競合を避けるために選択を無効にしつつ、依然として `elem` の内容はコピーできるようにします。

3. `document.getSelection().empty()` が発生したあと、事後の選択をクリアすることもできます。これは選択が表示されて消えるときに不要な点滅を引き起こすため、めったに使われません。

## リファレンス

- [DOM 仕様: Range](https://dom.spec.whatwg.org/#ranges)
- [Selection API](https://www.w3.org/TR/selection-api/#dom-globaleventhandlers-onselectstart)
- [HTML 仕様: テキストコントロール選択用 API](https://html.spec.whatwg.org/multipage/form-control-infrastructure.html#textFieldSelection)


## サマリ

選択のための２つの異なる API について説明しました:

1. document に対して: `Selection` と `Range` オブジェクト
2. `input`, `textarea` に対して: 追加のメソッドとプロパティ

２つ目のAPIはテキストに対してのみ動作するのでとてもシンプルです。

最もよく使用されるレシピはおそらく次のものです:

1. 選択を取得する:
    ```js run
    let selection = document.getSelection();

    let cloned = /* 選択したノードのクローンを作成する要素 */;

    // 次に selection.getRangeAt(0) に対して Range メソッドを適用します。
    // あるいは、ここのように複数選択をサポートするためにすべての範囲に適用します。
    for (let i = 0; i < selection.rangeCount; i++) {
      cloned.append(selection.getRangeAt(i).cloneContents());
    }
    ```
2. 選択を設定する:
    ```js run
    let selection = document.getSelection();

    // 直接:
    selection.setBaseAndExtent(...from...to...);

    // あるいは range　を作成し設定します:
    selection.removeAllRanges();
    selection.addRange(range);
    ```

そして、最後にカーソルについてです。`<textarea>` のような編集可能な要素のカーソル位置は常に選択範囲の先頭あるいは末尾になります。これを使用して、`elem.selectionStart` や `elem.selectionEnd` を設定することでカーソル位置を取得したりカーソルを移動させることができます。