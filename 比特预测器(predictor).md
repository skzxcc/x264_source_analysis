## 前言

在[码率控制-帧级码控](码率控制-帧级码控.md)、[码率控制-行级码控](码率控制-行级码控.md)两篇文章中都有提及到比特预测器，但没有展开讲。

所谓比特预测器就是根据现有的一些信息(如QP、SATD等)预测当前帧(宏块)编码后生成的比特大小，这部分的功能在x264整个码率控制模块尤为重要，因此将其单独作为一章进行讲解。(***比特预测器？ 姑且这么叫吧***)

**比特预测器？ 更学术性的说法是“建立Bit-QP之间的关系模型“**， 在x264里面确切的说是建立“**Bit-Qscale之间的关系模型**” （qscale与QP存在一对一的映射关系），所以下面的内容会直接用qscale来形容量化等级而不是QP



```c++
/* Terminology:
 * qp = h.264's quantizer
 * qscale = linearized quantizer = Lagrange multiplier
 */
// qp --->qscale关系式
static inline float qp2qscale( float qp )
{
    return 0.85f * powf( 2.0f, ( qp - (12.0f + QP_BD_OFFSET) ) / 6.0f );
}

// qscale ----> qp关系式
static inline float qscale2qp( float qscale )
{
    return (12.0f + QP_BD_OFFSET) + 6.0f * log2f( qscale/0.85f );
}
```



## 原理分析

***首先我们来思考一下，一帧图像编码后的生成比特数的大小与什么相关？***

我们肯定会想到与Qscale和图像复杂度相关，且Qscale越大比特数越小，复杂度越高比特数越大；即复杂度与比特数呈正相关，Qscale与比特数呈负相关。 

假设比特数为bits，Qscale为q，图像复杂度为var，我们先写出一个简单的关系：
$$
bits = α * var / q + β / q
$$
其中α和β是一个变量(有一个初始值)，**所谓预测器肯定不能保证实际比特数百分百符合预期的，它也是在不断学习、调整的，目的是让预估的比特数尽量符合实际的比特数**



### TEST:

以上述公式为例，假设当前帧的复杂度为var(在x264以sad或则satd表示)，帧级Qscale为q，预期的比特数为b1，实际的比特数为b2。

**正推：**b1 = α * var / q + β/q;

由于b2与b1的值可能存在一定偏差的，所以需要调整α和β的值，新的值用于下一次的比特预测。

 如何调整呢？**其实就是通过b2反推回去**

**反推：**

```tex
new_α = (b2 * q - β) / var; // b2、q、β、var都是已知变量,计算得到新的α
new_β = b2*q - new_α*var;   // b2、new_α、var、q都是已知变量,计算得到新的β
α = new_α;  // 更新α
β = new_β;  // 更新β
```



## 源码分析

我们将上述的bits = α * var / q + β/q公式稍微变化一下，变成
$$
bits = (coeff * var + offset) / (q * count)
$$
其中coeff /count可以认为就是α，offset/count就是β。



在x264涉及到predictor的函数不多，主要是**update_predictor**和**predict_size**



**predict_size**

```c++
// 根据复杂度、Qscale获取预测的比特数
static float predict_size( predictor_t *p, float q, float var )
{
    // 与上述公式一样
    return (p->coeff*var + p->offset) / (q*p->count);
}
```



**update_predictor**

```c++
// 更新预测器，就是上述说的更新α和β的值
static void update_predictor( predictor_t *p, float q, float var, float bits )
{
    float range = 1.5;
    if( var < 10 )
        return;
    float old_coeff = p->coeff / p->count;   // 获取旧的coeff
    float old_offset = p->offset / p->count;  // 获取旧的offset
    // 根据实际比特bits，反推回去得到新的coeff和offset
    float new_coeff = X264_MAX( (bits*q - old_offset) / var, p->coeff_min );
    // 防止更新前后的coeff相差太大，限制new_coeff的值
    float new_coeff_clipped = x264_clip3f( new_coeff, old_coeff/range, old_coeff*range );
    float new_offset = bits*q - new_coeff_clipped * var; // 得到新的offset
    if( new_offset >= 0 )
        new_coeff = new_coeff_clipped;
    else
        new_offset = 0;
    /*
    	这里x264的做法是没有直接将new_coeff和new_new_offset更新到coeff、offset
    	因为怕更新前后的coeff、offset值变化太大，毕竟比特预测器本身就不能保证很高的准确性
    	所以这里做个“平滑处理”，将原值乘上一个衰减因子后再加上新值
    */
    p->count  *= p->decay; // p->decay为衰减因子
    p->coeff  *= p->decay;
    p->offset *= p->decay;
    p->count  ++;
    p->coeff  += new_coeff;
    p->offset += new_offset;
}
```



[码率控制-帧级码控](码率控制-帧级码控.md)、[码率控制-行级码控](码率控制-行级码控.md)两个都有使用到predict_size、update_predictor。前者是针对一帧而言的，后者是一行而言的，两者是不一样的，所以x264中是维护了两个预测器分别给帧级、行级使用

```c++
/*
	截取部分代码
	x264_ratecontrol_new函数会在x264_encoder_open中被调用
*/
int x264_ratecontrol_new( x264_t *h )
{
    // ......
    int num_preds = h->param.b_sliced_threads * h->param.i_threads + 1;
    CHECKED_MALLOC( rc->pred, 5 * sizeof(predictor_t) * num_preds );
    CHECKED_MALLOC( rc->pred_b_from_p, sizeof(predictor_t) );
    static const float pred_coeff_table[3] = { 1.0, 1.0, 1.5 };
    for( int i = 0; i < 3; i++ )
    {
        rc->last_qscale_for[i] = qp2qscale( ABR_INIT_QP );
        rc->lmin[i] = qp2qscale( h->param.rc.i_qp_min );
        rc->lmax[i] = qp2qscale( h->param.rc.i_qp_max );
        for( int j = 0; j < num_preds; j++ )
        {
            // 初始帧级预测器的相关变量
            rc->pred[i+j*5].coeff_min = pred_coeff_table[i] / 2;
            rc->pred[i+j*5].coeff = pred_coeff_table[i];
            rc->pred[i+j*5].count = 1.0;
            rc->pred[i+j*5].decay = 0.5;
            rc->pred[i+j*5].offset = 0.0;
        }
        for( int j = 0; j < 2; j++ )
        {
            // 初始行级预测器的相关变量
            rc->row_preds[i][j].coeff_min = .25 / 4;
            rc->row_preds[i][j].coeff = .25;
            rc->row_preds[i][j].count = 1.0;
            rc->row_preds[i][j].decay = 0.5;
            rc->row_preds[i][j].offset = 0.0;
        }
    }
    rc->pred_b_from_p->coeff_min = 0.5 / 2;
    rc->pred_b_from_p->coeff = 0.5;
    rc->pred_b_from_p->count = 1.0;
    rc->pred_b_from_p->decay = 0.5;
    rc->pred_b_from_p->offset = 0.0;
    // ......
}
```





**Ref： https://github.com/mirror/x264**
