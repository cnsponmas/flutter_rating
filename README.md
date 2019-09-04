星级评分

因为工作中有一个需求是展示游戏评分，而Flutter系统并没有提供现成的评分控件，网上也能找到相关的自定义评分控件的文章，总是有点不符合我自己的需求，所以就有接下来的这篇文章了

### 关键点

关键是有三个点的效果要想好

1. 未选中星展示
2. 满星展示
3. 未满星展示

星际评分原理就是使用未选中星星作为背景，上层使用满星和未满星来覆盖展示，达到百分比的效果

1. 使用Stack层叠布局来控制两种样式的叠加
2. 使用裁剪CustomClipper来显示未满星的效果

### 具体实现

话不多说，上代码

我们可以提前设计一下提供给其他人使用时，可以自定义的内容，比如说星星个数、星星的间距、星星的大小等等，这里我把自己写的组件参数先提取了，你们可以根据自己的需要来扩展

```dart
class RatingBar extends StatefulWidget {
  final int count; 
  final double maxRating;
  final double value;
  final double size;
  final double padding;
  final String nomalImage;
  final String selectImage;
  final bool selectAble;
  final ValueChanged<String> onRatingUpdate;

  RatingBar({
    this.maxRating = 10.0,
    this.count = 5,
    this.value = 10.0,
    this.size = 20,
    this.nomalImage,
    this.selectImage,
    this.padding,
    this.selectAble = false,
    @required this.onRatingUpdate
  }) : assert(nomalImage != null),
        assert(selectImage != null);

  @override
  _RatingBarState createState() => _RatingBarState();
}
```

#### 1、背景星

我们只做横版效果，所以可以使用Row来做出星星的排列效果

```dart
List<Widget> buildNomalRow() {
    List<Widget> children = [];
    for(int i = 0; i < widget.count; i ++) {
      children.add(Image.asset(widget.nomalImage,height: widget.size,width: widget.size,));
      if(i < widget.count - 1) {
        children.add(SizedBox(width: widget.padding,));
      }
    }
    return children;
}
```

每个星星之间增加一个SizedBox作为间隔

#### 2、满星与未满星

我们根据评分和星星最大数量来计算每颗星星所对应的分数比例，然后根据分数比例来计算满星数量与未满星的裁剪比例

```dart
int fullStars() {
    if(value != null) {
      return (value /(widget.maxRating/widget.count)).floor();
    }
    return 0;
}

double star() {
    if(value != null) {
        if(widget.count / fullStars() == widget.maxRating / value ) {
        return 0;
        }
        return (value % (widget.maxRating/widget.count))/(widget.maxRating/widget.count);
    }
    return 0;
}

List<Widget> buildRow() {
    int full = fullStars();
    List<Widget> children = [];
    for(int i = 0; i < full; i ++) {
      children.add(Image.asset(widget.selectImage,height: widget.size,width: widget.size,));
      if(i < widget.count - 1) {
        children.add(SizedBox(width: widget.padding,),);
      }
    }
    if(full < widget.count) {
      children.add(ClipRect(
        clipper: SMClipper(rating: star() * widget.size),
        child: Image.asset(widget.selectImage,height: widget.size,width: widget.size),
      ));
    }
    return children;
}
```

> 满星个数计算：
>
> 每颗星星对应分数值 = 最大分值/星星数量  
>
> 当前评分 / 星星分数值 取整就是当前满星的个数

> 未满星裁剪比例：
>
> 当前评分制除以星星分数值取余，就是取出满星后的分数值，这个值再除以 星星分数值就是当前未满星星的裁剪比例

#### 裁剪

因为我们只需要竖向裁剪，所以用ClipRect就能满足需求了

```dart
class SMClipper extends CustomClipper<Rect>{
  final double rating;
  SMClipper({
    this.rating
  }): assert(rating != null);
  @override
  Rect getClip(Size size) {
    return Rect.fromLTRB(0.0, 0.0, rating , size.height);
  }

  @override
  bool shouldReclip(SMClipper oldClipper) {
    return rating != oldClipper.rating;
  }
}
```

传入裁剪比例，就能得到我们想要的未满星

#### 组合

两种星星样式都写好了，接下来组合。使用Statck层叠布局来保证两个样式的叠加

```dart
Widget buildRowRating() {
    return Container(
      child: Stack(
        children: <Widget>[
          Row(
            children: buildNomalRow(),
          ),
          Row(
            children: buildRow(),
          )
        ],
      ),
    );
  }
```

至此，一个静态的评分展示控件就已经写完了。

<img src='/Users/sponmas/Library/Application Support/typora-user-images/image-20190904175243007.png' width='40%'>

### 动态

既然后展示评分，那肯定就需要一个能评分的了，我们就需要在原有的基础上对控件做一个 触摸和点击事件监听，对结束触摸的点进行计算，来得到当前点代表的评分值

#### 监听移动

监听手势动作的话，我们就可以用到Listener了，这里需要用到它的两个回调方法onPointerMove和onPointerDown(或者onPointerUp)，前者用来监听滑动，后者用来监听点击

```dart
Listener(
      child: buildRowRating(),
      onPointerDown: (PointerDownEvent event){
        double x = event.localPosition.dx;
        if (x < 0) x = 0;
        pointValue(x);
      },
      onPointerMove: (PointerMoveEvent event) {
        double x = event.localPosition.dx;
        if (x < 0) x = 0;
        pointValue(x);
      },
      onPointerUp: (_) {
      },
      behavior: HitTestBehavior.deferToChild,
    )
```

根据手势的x坐标来计算当前手指位置代表的评分值，得到后触发重新构建，这里有个问题就是要去掉星星之间的间隔，保证精确

```dart
pointValue(double dx) {
    if(!widget.selectAble) {
      return;
    }
    if(dx >= widget.size * widget.count  + widget.padding * (widget.count - 1)) {
      value = widget.maxRating;
    }else {
      for(double i = 1; i < widget.count + 1;i ++) {
        if(dx > widget.size * i + widget.padding *(i -1) && dx < widget.size * i + widget.padding * i) {
          value = i * (widget.maxRating/widget.count);
          break;
        }else if(dx > widget.size * (i -1) + widget.padding*(i -1) && dx < widget.size * i+ widget.padding*i )  {
          value = (dx - widget.padding *(i -1))/(widget.size * widget.count ) *widget.maxRating;
          break;
        }
      }
    }
    setState(() {
      widget.onRatingUpdate(value.toStringAsFixed(1));
    });
  }
```

这样，一个可以动态评分的控件就完成了，怎么用呢，看下面

```dart
RatingBar(
    value: 9,
    size: 30,
     padding: 5,
     nomalImage: 'img/star_nomal.png',
     selectImage: 'img/star.png',
     selectAble: true,
     onRatingUpdate: (value) {},
     maxRating: 10,
     count: 6,
     )
```

各参数说明

- value：当前评分值
- size：星星大小
- padding：星星间距
- nomalImage：空星图片
- selectImage：满星图片
- selectAble：是否可以点击滑动修改评分值
- onRatingUpdate：点击滑动修改评分值回调，参数是String类型的评分值
- maxRating：最大评分值
- count：星星个数

