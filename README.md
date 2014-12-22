Unity で高品質な減色を実現する
==============================

概要
----

Unity ではテクスチャ画像のフォーマットに 16 bit color を指定すると単純なビットシフトによる減色が行われますが、これは画質の大きな劣化を招きます。この文書では、ディザリングアルゴリズムを使ってこれを改善する方法を紹介します。

動機
----

Android では機種に依存せず使用可能なテクスチャ圧縮アルゴリズムとして ETC1 が採用されていますが、この ETC1 ではアルファチャンネルを用いることができません。そのため、Android においてアルファを用いたい場合には、画質の悪化を我慢して 16 bit color を適用するか、メモリ不足のリスクを承知で 32 bit color を適用するか、どちらか２択を強いられることになります。

（もちろん、GPU コア毎に特化されたテクスチャ圧縮形式を適用することも可能ですが、機種毎に個別のパッケージを用意する手間を考えると、なかなか採用しづらいというのが実際のところです）

Unity において 16 bit color の画質が著しく悪化する原因は、その減色方式の単純さにあります。単なるビットシフトにより各色成分の下位 4 bit を切り捨てるという処理が行われるため、繊細なグラデーションはすべて潰れて顕著なマッハバンドを残すことになります。

たとえ単純なアルゴリズムでも、何らかのディザリングを適用すれば、そこまで悪い画質にはならないはずです。そこで、テクスチャインポート時に簡単なディザリングを適用するスクリプトを組んでみることにしました。

デフォルトの 16 bit 減色例
--------------------------

左がオリジナルの画像、右が 16 bit 減色後の画像です。グラデーションが潰れてマッハバンドが現れているのが見えます。肌の色も若干異なって見えます。

![Image A (original)](http://keijiro.github.io/unity-dither4444/a-original.png)![Image A (default)](http://keijiro.github.io/unity-dither4444/a-default.png)

もう一例挙げます。

![Image B (original)](http://keijiro.github.io/unity-dither4444/b-original.png)![Image B (default)](http://keijiro.github.io/unity-dither4444/b-default.png)


ディザリング適用時の 16 bit 減色例
----------------------------------

この文書で提示する方法によりディザリングを適用した例です。左がオリジナルの画像、右が 16 bit 減色後の画像です。若干ジャリジャリとしたノイズが混ざる形となっていますが、全体としてマッハバンドは軽減されていることが分かります。

![Image A (original)](http://keijiro.github.io/unity-dither4444/a-original.png)![Image A (dithered)](http://keijiro.github.io/unity-dither4444/a-dither.png)

![Image B (original)](http://keijiro.github.io/unity-dither4444/b-original.png)![Image B (dithered)](http://keijiro.github.io/unity-dither4444/b-dither.png)

![Image C (original)](http://keijiro.github.io/unity-dither4444/c-original.png)![Image C (dithered)](http://keijiro.github.io/unity-dither4444/c-dither.png)

![Image D (original)](http://keijiro.github.io/unity-dither4444/d-original.png)![Image D (dithered)](http://keijiro.github.io/unity-dither4444/d-dither.png)

スクリプトの説明
----------------

減色処理のメインとなるのは [Assets/Editor/TextureModifier.cs](https://github.com/keijiro/unity-dither4444/blob/master/Assets/Editor/TextureModifier.cs) です。実装の詳細については、こちらのスクリプトを参照してください。

このスクリプトでは、AssetPostprocessor を用いてテクスチャのインポート処理をオーバーライドしています。今回のサンプルでは、とりあえずファイル名の末尾が “Dither” となっていた場合に、自動的にディザリングを施すようにしました。

大まかには次のような実装になっています。

#### OnPreprocessTexture

テクスチャのインポートが開始される前に呼ばれる関数です。ここでは、ターゲットがディザリング対象のテクスチャであった場合に、テクスチャのフォーマットを RGBA32 に変更する処理を行っています。これは、後に続く処理の中で減色前のデータを正しく拾うために必要になります。

また、このサンプルではすべてのテクスチャが GUI 用途であるため、テクスチャタイプを “GUI” に変更する処理も行っています。これは簡便化のために挟んでいるだけで、本来は不要な処理です。

#### OnPostprocessTexture

テクスチャのインポートが終了した後に呼ばれる関数です。ここでは、ターゲットがディザリング対象のテクスチャであった場合に、以下の処理を行います。

1. ディザリングの適用。
2. RGBA4444 形式への変換。

まず最初にディザリングを行います。元画像を float color （Color クラス）列で取得し、ディザリングと 4 bit クオンタイズ（量子化）を行います。この時点で各色成分は 4 bit 精度にまで落ちていますが、結果は float color のままターゲットのテクスチャへ再格納します。

次に、ターゲットのテクスチャを RGBA4444 の 16 bit テクスチャ形式へ変換します。これには EditorUtility.CompressTexture を用います。これにより、結果的にディザリング適用済みの画像が 16 bit 形式で格納されることになります。

今後の課題
----------

このサンプルではシンプルな [Floyd-Steinberg のディザリングアルゴリズム](http://en.wikipedia.org/wiki/Floyd–Steinberg_dithering) を用いていますが、このアルゴリズムは必ずしも高品質な結果をもたらしません。もっと複雑なディザリングアルゴリズムを用いることで、より高い品質を得ることができるかもしれません。

また、単純に RGB 色空間上で処理を行っていることも、結果に悪影響を及ぼしているように思われます。微妙なグラデーション部分に見られる色相の乱れなどは、一時的な色空間の変換を行うことで改善が可能かもしれません。

謝辞（テスト画像について）
--------------------------

上でテストに用いている画像 (Test A, Test B) は[テラシュールウェア](http://terasur.blog.fc2.com)さんよりご提供いただいたものです。 **この画像を検証以外に用いることは避けてください。**

スクリプトの使用について
------------------------

このプロジェクトに含まれるスクリプト (TextureModifier.cs) は商用・非商用を問わず自由に利用して構いません。
