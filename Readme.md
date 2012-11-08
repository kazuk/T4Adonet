#Wellcome T4 ADO.NET 

T4 により SQL Server 相手にSQLコマンドをスキーマ取得モードで発行し、結果をもとに ADO.NET でのデータフェッチコードを生成します。

生成コードは基本的には Patterns & Practices の ADO.NET のパフォーマンスとスケーラビリティに関するドキュメントを元にここまでやればスピード狂のニーズにも十分という線引きで実装しています。

Chapter 12 - Improving ADO.NET Performance and Scalability 
http://msdn.microsoft.com/en-us/library/ff647768.aspx

SelectCommandGenerator.tt がメインのソースで、この冒頭に設定するべき項目が用意されています。
SelectCommandGenerator.cs がサンプルとしての出力になります。

## 設定

### connectionString 変数
  このttがスキーマ情報を取得する元となる SQL Server への接続文字列を記述します。
  すべてのコマンドは CommandBehavior.SchemaOnly で発行されるため、実際には実行されません。

コマンド文字列リテラル
  ToLiteral によってT4の出力に設定された文字列をディクショナリに取得できます。
  当然にT4の出力文字列部では特にコード上で必要となるエンコード等は不要ですので、生のSQL文をべったりと貼り付けてください。

```
using( ToLiteral(literalContainer, "QueryName") ) {#>
SQL文
<#	}
```

　上記の場合には QueryName キーに SQL 文が格納されます。

### commands 変数
　生成するコマンドをSelectCommandDefinition クラスの配列に一つ以上設定します。
* ClassName プロパティ 生成されるクラス名を指定します。
* CommandText プロパティ コマンドのテキストを設定します。
* Parameters プロパティ コマンドのパラメータリストを設定します。

### namespaceName 変数
　生成されるクラス群を配置する名前空間名を指定します。

### GCOptimizeForStringColumn プロパティ
　クラス名、結果セットインデックス、カラム名を元に以下の GC 最適化を適用するか否かを判別するラムダを設定します。

　ラムダが false を返した場合、最適化は行われず、文字列カラムのプロパティは通常の System.String 変数が使われます。
ラムダが true を返した場合、最適化が行われ、文字列カラムのプロパティは StringBuffer 構造体のインスタンスになります。
StringBuffer構造体は char配列により文字列の内容を保持し、長さのint値と共に保持されます。
StringBuffer構造体に設定されたchar配列は繰り返し再利用されますので、アプリケーションでこの参照を保持しないようにして
下さい。必要な場合には内容をどこかに複製してください。

NOTE: 
  現状では各コマンドのクラスのスコープ内で StringBuffer が宣言されます。この挙動は拡張メソッドを定義する場合に非常に困ったことになりますので変更する予定です。またバッファの管理が現状では生成クラス中に閉じていますので、多数のクラスで使用した場合にバッファの使用率がイマイチのはずです。この辺は余りにグローバルなバッファマネージャを実装するとスレッドの競合等で性能が落ちる懸念からこのような作りになっていますが、各スレッド毎のメモリプールを正しくハンドルできる様にする方向で考えています。



## 呼び出し側の実装

　単純に new して Execute して Fetch するという事です。

```cs
  using( SqlConnection conn = new SqlConnection( connectionString ) )
  using( GeneratedCommand cmd = new GeneratedCommand( conn ) )
  {
      conn.Open();
	  try {
	      using( var reader = cmd.Execute( parameter ) ) {
	          foreach( var record in cmd.Fetch0( reader ) ) { process it }
	          reader.NextResult();
	          foreach( var record in cmd.Fetch1( reader ) ) { process it }
			  ... to be continue
			  reader.Close();
		  }
	  }
	  finally {
	     conn.Close()
	  }
  }
```

バッチコマンドに対応しています。
サンプルの生成内容通りですが、バッチコマンドに対応しています。複数のコマンドを単独のSQLコマンドとして発行でき、結果セットの取得には Fetch0～FetchN で順番に結果セットをフェッチします。


生成されるコマンドクラスは SqlCommandクラスの薄いラッパーであり、SqlDataReaderからいかに高速にデータをフェッチするか、および無駄なメモリ消費を抑える事を目的にしています。
内部で ConcurrentStack に基づいた生成済み SqlCommand のキャッシュを行うため、繰り返し利用されるコマンドで最良のパフォーマンスを発揮するようになっています。

キャッシュがらみのライフサイクル管理のために生成クラスは IDisposable になっています。Dispose が呼び出されると内部で保持しているSqlCommandのConnectionプロパティにnullを設定する事でSqlConnectionから切り離し、キャッシュに格納します。
堅牢性という観点ではキャッシュに収容されたインスタンスの最大数を管理するべきですが、以下の理由から行っていません。
 Disposeを呼び出さなかった場合にはキャッシュが枯渇するだけですのでSqlCommandを再生成するコストがかかるだけで正常に動作します。
 生成側を呼び出さなかった場合には使われないわけですので、性能影響はありません。


