# ChloroplastAnnotation
叶绿体基因组注释流程
使用软件自动注释的结果不仅不符合NCBI的格式要求，而且一般还会有生物学问题，故记录下处理流程。  

**更新日期：2022-01-25**  
**更新内容：**  
----------2022-01-25----------
1. curated_geseq.py增加了auto选项：是否自动在输出的GeSeq gff文件中将检查有问题的条目删除/修改（修改详见2）
2. check中检查基因名在检查的基础上可以进行一定程度的自动修订（如clpP1修正为clpP）
3. 常见存在重叠区域的基因将不再提示（如matK和trnK-UUU），减少人工审核工作量
4. 进一步优化命令行参数
5. pre_check步骤不再需要先做PGA注释，改为由pre_check.py直接调用BLAST完成，更加便捷（详见步骤1）
6. pre_check在基于trnH的基础上进一步利用psbA的信息，避免了部分叶绿体基因组有两个trnH带来的误判
7. 重写文档与实例文件

----------2022-01-21----------
1. 重写大部分脚本，大幅缩减了代码量。gffutils编写的工具已全部用GFF重写。
2. 完善了reference文件
3. 修正了校正与插补功能中的bug



## 0. 整体流程

1. 校正序列
2. 注释校正后的序列(PGA+GeSeq)
3. **GeSeq结果校正**
4. 自动检查与校正
5. **人工检查**
6. 重新排序并添加rps12

 *ps:* 加粗的步骤表示需要手工处理

## 1.校正序列
主要涉及以下两个方面
### 检查保守基因
检查保守基因是为了确保序列组装的正确性。本流程使用matK和rbcL作为保守基因进行检查，若组装结果缺少这两个基因中的任何一个，请考虑组装结果的正确性。
*ps:* 缺失也不一定错误，如南方菟丝子（寄生植物）就是没有matK
### 校正序列
校正序列主要涉及三个内容
1. 序列的链  
虽然NCBI等数据库对具体使用哪条链上传数据并无强制性要求，但是建议使用常用的链作为序列文件，即trnH的反义链，这样有利于后续分析
(比如GeSeq注释如果链不对的话有时会注释不出来，PGA没有这个问题)。
2. 序列的起始位置  
校正trnH至序列起始位置，这是因为叶绿体基因组为环状序列，fasta文件中的开头和结尾是人为断开的，可能存在跨越了这个gap 的基因。校正trnH至起始位置可以有效避免出现跨越开头-结尾gap的基因。
3. 序列中的模糊碱基
当序列中有ATCGN(atcgn)以外的碱基时，会在log中进行提示。**注意，不会自动校正**

~~~bash
python pre_check.py -i  raw_seq.lst -o /path/to/fasta > pre_check.log
# raw_seq.lst 每行一个fasta路径
~~~

## 2.使用GeSeq、PGA注释校正后的序列
### GeSeq
[GeSeq](https://chlorobox.mpimp-golm.mpg.de/geseq.html)是目前主流的注释软件之一。首先使用GeSeq进行注释。其中，与已知数据库的比对，蛋白编码基因BLAT设置阈值为比对identity 大于85%，rRNA/tRNA基因BLAT设置阈值为比对identity大于85%。**并从同科的物种中额外添加一些作为注释参考（原因见PGA软件说明）。** 需要注意的是，不要勾选其他注释选项，如Chloë。   
GeSeq注释结果的主要问题是基因命名极不规范，例如"gene-blatn_trnI-CAU_1"；其次是gene和CDS、tRNA等之间的Parent-Child关系缺失。
因此会在步骤6中进行转换

### PGA
[PGA](https://github.com/quxiaojian/PGA)也是目前主流的注释软件之一。使用PGA进行注释，相似度阈值设置为85%。**参考物种只选择该植物类型的参考物种（如被子植物选择无油樟，裸子植物使用美叶苏铁，蕨类植物选择蛇足石杉）**  

~~~shell
while read seq_name
do
    perl /path/to/PGA/PGA.pl \
          -r /path/to/ref/Angiosperm \
          -p 85 \
          -i 100000 \
          -t /annotation/angiosperm/seq/${seq_name}.fasta \
          -o /annotation/angiosperm/gb > /dev/null
    mv /path/to/annotation/angiosperm/gb/${seq_name}.gb /path/to/annotation/angiosperm/gb2/${seq_name}.gb
done
~~~
*ps1.* 反向重复区最短长度设为10k是因为我不想让PGA注释反向重复区，具体原因感兴趣的话可以阅读[开发文档](Development.md)  
*ps2.* 用mv命令是因为PGA每次会自动覆盖上一个output文件夹，而我不想搞出一堆文件夹，毕竟我只要genbank文件。

### 说明
使用两个软件注释的原因是一个软件注释不准确（例如起始、终止密码子错误）。但是要注意，两个软件使用的参考物种是不同的，这是出于以下原因：如果只使用基部物种（如无油樟），则一些在近源种中才有的基因会无法注释到;如果选择了相应的近源物种，则在实际操作中会出现基因名混乱的问题。这是因为现有软件都是基于BLAST进行注释的，所以如果参考基因组中的基因名是错误的话，则注释出来的也是错误的，如参考基因组中如果错误的将atpI写成了aptI，trnK-UUU简写成trnK，则注释结果中就会出现这两种名称。显然这是不对的。基于此，我在使用两种软件进行注释时分别选取了不同的策略，从而保证注释全面、名称准确。而之所以选择GeSeq添加近源物种、PGA只使用基部物种，是因为GeSeq作为在线服务器选择近源物种较方便，同时其基因名本身就不规范（主要它的那个Reference没有公开，不知道具体内容），后期需要校正，校正一个也是校，多校正一个近源物种的也是校。另一方面，PGA对于参考文件的要求较高（见FAQ2）,每个物种的近源种都制作的话太过耗费时间。

## 3.GeSeq结果校正
使用[curate_geseq.py](curate_geseq.py)将GeSeq的注释结果转换。  
geseq_info,txt, pga_info.txt和geseq.log请参考[example](example)文件中的示例。  
**注意：PGA和GeSeq转换时请使用不同的前缀，否则后面的校正解析时会出问题，具体原因感兴趣的话可以阅读[开发文档](Development.md)**
~~~bash
python curated_geseq.py -i geseq.lst -o /path/to/curated -a > geseq.log
# geseq.lst  每行一个GeSeq输出文件
~~~

转换完成后，根据log文件，手工校正每一个GeSeq中的不规范名称。**这步非常重要，因为后面的校正脚本中很多是依赖基因名的，所以必须认真。** 该步骤非常耗时间，为了节省时间，增加了`-a`选项，使用该选项的话，所有geseq.log中基因名有问题的基因都会先自动校正（如clpP1校正为clpP，ndha校正为ndhA），无法自动校正的会丢弃（比如XXX-fragment这种）。  
*ps.* PGA结果不需要手工校正的原因是PGA注释时只使用了基部植物的参考基因组，基因名已经人工校正过，所以结果会非常整齐。如果你PGA注释也使用了自己添加的参考物种的话，那么这边同样需要手工校正。

## 4. 自动检查与校正
基于步骤6生成的两个gff注释文件，使用[correct.py](correct.py)进行校正  
correct_info,txt和correct.log请参考[example](example)文件中的示例。
~~~bash
python correct.py combine -i correct_info.txt -o correct > correct.log
# correct_info.txt包含三列（请不要保留列名）
# geseq转换后gff的路径    PGA genbank文件路径    基因前缀
~~~

目前自动检查、校正实现了以下功能：
1. 起始密码子检查(目前对于起始密码子中存在的RNA编辑现象，还未决定最终如何处理，暂且当做合法密码子处理，例如ACG->AUG)
2. 终止密码子检查
3. 序列中是否有终止密码子
4. 序列长度是否为3的倍数
5. CDS是否过短（小于33bp)
5. tRNA位置信息是否正确
6. 是否存在tRNA缺失(GeSeq的注释有时候会莫名其妙少很多tRNA)，及自动插补
7. 基因组区域是否存在重叠
8. 起始密码子可能存在RNA-editing现象的基因，自动在起始密码子后30bp寻找替代起始密码子。

*注*：
1. 以上功能均为自动实现，对于两个GeSeq和PGA注释都有问题的基因（例如根据两个注释gff文件，其实密码子都有问题），会输出到log中，并且在gff文件中该基因的attributes会标注"pseudo=true"
2. 由于rps12基因的特殊性，所有检查均不涉及rps12基因（脚本检测到rps12基因会自动跳过）
3. 由于计算机计数从0开始和python的切片机制，因此log文件中位置信息的起始位点会和gff文件中的相差1，是正常现象，不是错误。

## 5. 人工检查
根据步骤4的log文件检查每个gff文件中区域重复的基因，并且删去相应的重复条目（包括gene和其附属的CDS、exon等。常见的一些重复区域将不会report，减少工作量）。删去cds区长度小于33的基因。同时检查pseudo gene是否需要保留，详细指导见FAQ3.

## 6.重新标号与自动添加rps12基因
由于5中删去了一些基因条目，所以需要对重新排序，会出现否则明明只有124个基因，但是最后一个基因名却是ABMAT127会给人造成误解（从文件格式上来说并没有错，但是我强迫症）。另一方面是上述所有步骤都屏蔽了rps12基因，因为这个基因是trans-spliced基因，处理起来非常麻烦，所以放在最后加。  
*ps:* 目前自动rps12基因只限于高等植物，这是因为低等植物和高等植物的rps12有些不一样，具体原因感兴趣的话可以阅读[开发文档](Development.md),后期可能会完善这个问题。

~~~bash
python correct.py rps12 -i rps12_info.txt > rps12.log
# rps12_info.txt包含四列（请不要保留无列名）
# 待处理gff的路径    结果gff存储路径    PGA原始genbank文件路径     基因前缀

~~~

## 9. （可选）生成NCBI上传所需文件
如果需要上传NCBI，请使用[gff2tbl.py](utilities/gff2tbl.py)转换完全正确的gff至tbl文件。由于rps12的特殊性，该基因的信息需要手动添加。

## FAQ
1. 不规范命名基因  
根据目前处理数据的经验，一般有以下几种情况：  
1.1 大小写问题  
如rbcl和rbcL。这种问题一般是参考基因组里不规范命名引起的。所以如果有很多序列使用同一个参考基因组注释，建议先人工核对参考基因组。  
1.2 异名问题  
如lhbA实际上为psbZ（目前版本针对常见异名已经做了校正处理）  
1.3 注释不完整  
一般有fragment字样，这种一般不需要再进行处理，就是要注意tRNA的fragment的product会统一被标注为hypothetical protein，需要手动
修改。  
不规范命名基因产生的原因就是参考基因组上命名不规范，在步骤5的说明中已经详细阐述。本流程认可的标准基因名请见[ref](ref)中的rna.txt
和cds.txt

2. PGA参考基因组的说明  
虽然PGA的作者只表示参考基因组应当手工校正，但是没有提出明确的标准，根据我在使用中的经验，参考基因组应该要符合以下方面的要求  
2.1. 规范的基因名  
2.2. 起始和终止密码子  
起始和终止密码子不确定或缺失的话，基因是无法正确被PGA注释的。Genbank文件中，一般这种情况的条目是通过在起始/终止位置添加\<或\>符号来表示的（具体与链和转录的方向有关，可以查看NCBI相关说明文档），所以碰到这种条目必须把相应位置的碱基替换一下换成合法的密码子，并将Genbank条目中的这些符号删去。如此才能正确注释。  
*ps:* 这里替换碱基不涉及数据造假，因为这里是只是为了做为注释参考而构建一个正确的基因集。换句话说修改后的参考基因组只能用于注释，
不能用于进化分析。  
2.3 基因应该尽量的全  
被子植物和裸子植物的参考基因组是PGA作者选定并校正过了的，囊括了所有主要的基因。蕨类植物的我自己找了蛇足石杉并校正了，在ref文件
夹中。

3. 重复区域的说明  
叶绿体基因组基因密度非常高，所以基因发生区域重叠不一定是错误，常见的存在区域重叠的基因见[ref](ref)中的region.json
3.1 tRNA与编码基因重复  
典型的就是trnK-UUU和matK。这一类情况主要是tRNA存在内含子，然后编码基因正好落在内含子区域。部分编码基因也存在这种情况（如petB
在有的物种的注释中就有这种情况）  
3.2 编码基因之间重复  
一般是在基因开头或末尾有几bp到一二十bp不等的重复。如atpB和atpE。  
一般需要校正的其实是完全重复区域的基因，如`['trnM-CAU'] [53865,53936]  and  ['trnT-GGU'] [53870,53928]  are duplicated`和
`['rrn5'] [107057,107177]  and  ['rrn5S'] [107057,107187]  are duplicated`这种情况。
   
3. 为什么correct步骤中有同一个基因出现两次的情况？  
部分基因组的correct.log可能会出现类似以下的信息，其中`ndhD`和`ycf15`重复了两次。这是因为GeSeq注释中`ndhD`出现了两次，而这两个基因的CDS检查都未通过，所以自动插补了PGA注释中的`ndhD`。而PGA中只有一个`ndhD`，所以是一个基因被插补了两次，因此造成了重复。
~~~shell
trnI-CAU [95742,95816]  and  ycf2 [95812,102751]  are duplicated
ndhD [133416,134919]  and  ndhD [133416,134919]  are duplicated
ycf15 [182201,182456]  and  ycf15 [182201,182456]  are duplicated
ycf2 [182604,189543]  and  trnI-CAU [189539,189613]  are duplicated
check duplicated region done
~~~

5. RNA-editing  
RNA editing是指从参考基因组转录到RNA的生物过程是由一个特殊的酶完成的，从而使得碱基发生了替换，如下图所示。
![wxZPp9.png](https://s1.ax1x.com/2020/09/23/wxZPp9.png)  
需要注意的是，在基因组注释过程中，除非有直接的转录组证据，否则ACG不应当直接注释为RNA-editing，而是需要注释为pseudogene.

## 仍待完善的功能和已知Bug
1. 步骤6中tRNA名称校正未实现自动化
2. rps12目前只能针对高等植物，低等植物的还不行
