#### 🐱‍👤TODO：本篇幅待整理，目前只贴出代码以及相关注释； 后续会补充说明MB-Tree相关原理，方面代码的理解



## 前言

**这里只分析none-B frame的情况**











```c++

static void macroblock_tree( x264_t *h, x264_mb_analysis_t *a, x264_frame_t **frames, int num_frames, int b_intra )
{
    int idx = !b_intra;
    int last_nonb, cur_nonb = 1;
    int bframes = 0;

    x264_emms();
    float total_duration = 0.0;
    for( int j = 0; j <= num_frames; j++ )
        total_duration += frames[j]->f_duration;
    float average_duration = total_duration / (num_frames + 1);

    int i = num_frames; // num_frames表示用于分析的帧数量(即lookahead的帧数)

    if( b_intra )
        slicetype_frame_cost( h, a, frames, 0, 0, 0 ); // 如果当前帧是关键帧则计算其intra cost(如果之前计算过这里不会重复计算)

    while( i > 0 && IS_X264_TYPE_B( frames[i]->i_type ) )
        i--;
    last_nonb = i;

    /* Lookaheadless MB-tree is not a theoretically distinct case; the same extrapolation could
     * be applied to the end of a lookahead buffer of any size.  However, it's most needed when
     * lookahead=0, so that's what's currently implemented. */
    if( !h->param.rc.i_lookahead )
    {
        if( b_intra )
        {
            memset( frames[0]->i_propagate_cost, 0, h->mb.i_mb_count * sizeof(uint16_t) );
            memcpy( frames[0]->f_qp_offset, frames[0]->f_qp_offset_aq, h->mb.i_mb_count * sizeof(float) );
            return;
        }
        XCHG( uint16_t*, frames[last_nonb]->i_propagate_cost, frames[0]->i_propagate_cost );
        memset( frames[0]->i_propagate_cost, 0, h->mb.i_mb_count * sizeof(uint16_t) );
    }
    else
    {
        if( last_nonb < idx )
            return;
        memset( frames[last_nonb]->i_propagate_cost, 0, h->mb.i_mb_count * sizeof(uint16_t) );
    }
	
    /*
    	功能：从lookahead缓存的最后一帧逐帧往前分析，这里主要是为了计算当前帧各个宏块的遗传价值
    	(如果宏块的遗传价值越大说明对于后续帧的作用越大，此时应该分配更低的QP去保其质量)
    	
    	问题：为什么需要从最后一帧逐帧往前分析呢？(当单一参考帧的情况下当前帧只会被下一帧作为参加帧，不是只分析下一帧就可以吗？)
    	解答：这里就体现了遗传价值的“遗传性”，举个例子：[帧0的MB_0]被[帧1的MB_0]参考，而[帧1的MB_0]被[帧2的MB_1]参考; 此时[帧0的MB_0]对[帧1的MB_0]的“贡献”会间接影响到[帧2的MB_1], 因此计算[帧0的MB_0]遗传价值不仅仅需要考虑[帧1的MB_0]还需要考虑到[帧2的MB_1]....(直到[帧x的MB_x])，所以需要从最后一帧逐帧往前分析
    		
    */
    while( i-- > idx ) 
    {
        cur_nonb = i;
        while( IS_X264_TYPE_B( frames[cur_nonb]->i_type ) && cur_nonb > 0 )
            cur_nonb--;
        if( cur_nonb < idx )
            break;
        // 分析last_nonb与cur_nonb的编码代价,non-B帧的情况下,cur_nonb为last_nonb-1，即前向参考前一帧进行分析
        slicetype_frame_cost( h, a, frames, cur_nonb, last_nonb, last_nonb );
        memset( frames[cur_nonb]->i_propagate_cost, 0, h->mb.i_mb_count * sizeof(uint16_t) );
        bframes = last_nonb - cur_nonb - 1;
        if( h->param.i_bframe_pyramid && bframes > 1 ) // 存在B帧的情况
        {
            int middle = (bframes + 1)/2 + cur_nonb;
            slicetype_frame_cost( h, a, frames, cur_nonb, last_nonb, middle );
            memset( frames[middle]->i_propagate_cost, 0, h->mb.i_mb_count * sizeof(uint16_t) );
            while( i > cur_nonb )
            {
                int p0 = i > middle ? middle : cur_nonb;
                int p1 = i < middle ? middle : last_nonb;
                if( i != middle )
                {
                    slicetype_frame_cost( h, a, frames, p0, p1, i );
                    macroblock_tree_propagate( h, frames, average_duration, p0, p1, i, 0 );
                }
                i--;
            }
            macroblock_tree_propagate( h, frames, average_duration, cur_nonb, last_nonb, middle, 1 );
        }
        else
        {
            while( i > cur_nonb ) // non-B的情况下不会进入到下面
            {
                slicetype_frame_cost( h, a, frames, cur_nonb, last_nonb, i );
                macroblock_tree_propagate( h, frames, average_duration, cur_nonb, last_nonb, i, 0 );
                i--;
            }
        }
        // 重点函数： 遗传价值分析(计算cur_nonb每个宏块的遗传价值)
        macroblock_tree_propagate( h, frames, average_duration, cur_nonb, last_nonb, last_nonb, 1 );
        last_nonb = cur_nonb; // 更新last_nonb，逐帧往前分析
    }

    if( !h->param.rc.i_lookahead ) // 如果没有lookahead
    {
        slicetype_frame_cost( h, a, frames, 0, last_nonb, last_nonb );
        macroblock_tree_propagate( h, frames, average_duration, 0, last_nonb, last_nonb, 1 );
        XCHG( uint16_t*, frames[last_nonb]->i_propagate_cost, frames[0]->i_propagate_cost );
    }
	// 到这一步，当前帧所有宏块的遗传价值已经计算完了，macroblock_tree_finish就是根据遗传价值去设置宏块的QP offset
    macroblock_tree_finish( h, frames[last_nonb], average_duration, last_nonb );
    if( h->param.i_bframe_pyramid && bframes > 1 && !h->param.rc.i_vbv_buffer_size )
        macroblock_tree_finish( h, frames[last_nonb+(bframes+1)/2], average_duration, 0 );
}
```





```c++
/*
	功能：计算frames[p0]每个宏块的遗传价值
	说明：
		为了计算frames[p0]的帧遗传价值，需要依赖frames[b]本身的遗传价值， 即：
		p0的遗传价值 = b帧本身对后续帧的遗传价值 + b帧继承自p0的价值
		
*/

static void macroblock_tree_propagate( x264_t *h, x264_frame_t **frames, float average_duration, int p0, int p1, int b, int referenced )
{
    // ref_costs[0]前向帧的遗传价值指针， ref_costs[1]为后向帧的暂不分析
    uint16_t *ref_costs[2] = {frames[p0]->i_propagate_cost,frames[p1]->i_propagate_cost};
    // 距离因子，在non-B帧的情况下这个值一般都是一样的
    int dist_scale_factor = ( ((b-p0) << 8) + ((p1-p0) >> 1) ) / (p1-p0);
    // 加权预测的权重？
    int i_bipred_weight = h->param.analyse.b_weighted_bipred ? 64 - (dist_scale_factor>>2) : 32;
    int16_t (*mvs[2])[2] = { b != p0 ? frames[b]->lowres_mvs[0][b-p0-1] : NULL, b != p1 ? frames[b]->lowres_mvs[1][p1-b-1] : NULL };
    int bipred_weights[2] = {i_bipred_weight, 64 - i_bipred_weight};
    // buf指向一块内存空间，用于保存frames[b]整合的遗传价值(包括b帧本身对后续帧的遗传价值 + b帧继承自p0的价值)
    int16_t *buf = h->scratch_buffer;
    uint16_t *propagate_cost = frames[b]->i_propagate_cost; // frames[b]对于lookahead中后续帧的遗传价值
    uint16_t *lowres_costs = frames[b]->lowres_costs[b-p0][p1-b]; // frames[b]各个宏块的编码代价，在前面slicetype_mb_cost中已经计算过了

    x264_emms();
    float fps_factor = CLIP_DURATION(frames[b]->f_duration) / (CLIP_DURATION(average_duration) * 256.0f) * MBTREE_PRECISION;

    /* For non-reffed frames the source costs are always zero, so just memset one row and re-use it. */
    if( !referenced )
        memset( frames[b]->i_propagate_cost, 0, h->mb.i_mb_width * sizeof(uint16_t) );

    for( h->mb.i_mb_y = 0; h->mb.i_mb_y < h->mb.i_mb_height; h->mb.i_mb_y++ ) // 
    {
        int mb_index = h->mb.i_mb_y*h->mb.i_mb_stride;
        // 整合frames[b]的遗传价值(包括本身对于lookahead后续帧的价值 + 本身有多少价值是继承自p0的)
        h->mc.mbtree_propagate_cost( buf, propagate_cost,
            frames[b]->i_intra_cost+mb_index, lowres_costs+mb_index,
            frames[b]->i_inv_qscale_factor+mb_index, &fps_factor, h->mb.i_mb_width );
        if( referenced )
            propagate_cost += h->mb.i_mb_width;
		/*
			根据slicetype_mb_cost中得到的frames[b]各个宏块的运动矢量以及mbtree_propagate_cost计算到的frames[b]各个宏块的遗传价值
			来计算frames[p0]每个宏块的遗传价值
		*/
        h->mc.mbtree_propagate_list( h, ref_costs[0], &mvs[0][mb_index], buf, &lowres_costs[mb_index],
                                     bipred_weights[0], h->mb.i_mb_y, h->mb.i_mb_width, 0 );
        if( b != p1 )
        {
            // 存在B帧的情况
            h->mc.mbtree_propagate_list( h, ref_costs[1], &mvs[1][mb_index], buf, &lowres_costs[mb_index],
                                         bipred_weights[1], h->mb.i_mb_y, h->mb.i_mb_width, 1 );
        }
    }
	/*
    	根据遗传代价计算frames[b]各个宏块的QP Offset
    	其实每个帧在macroblock_tree中的macroblock_tree_finish中都会重新计算一次Qp offset，所以这里的macroblock_tree_finish看起来好像没什么用？？？
    */
    if( h->param.rc.i_vbv_buffer_size && h->param.rc.i_lookahead && referenced )
        macroblock_tree_finish( h, frames[b], average_duration, b == p1 ? b - p0 : 0 ); 
}
```





```c++
/* Estimate the total amount of influence on future quality that could be had if we
 * were to improve the reference samples used to inter predict any given macroblock. */
/*
	作用：整合遗传价值(包括frames[b]帧本身对后续帧的遗传价值 + frames[b]帧继承自p0的价值)
	
	dst: 用于保存整合的遗传价值
	propagate_in：frames[b]帧本身对后续帧的遗传价值(这个在上一次frames[b]、frames[b+1]就已经得到了)
	intra_costs：frames[b]帧内编码代价(在前面的slicetype_mb_cost计算得到)
    inter_costs：frames[b]帧间编码代价(在前面的slicetype_mb_cost计算得到)
    inv_qscales： 放大因子(这里的值在AQ中确定，与AQ中计算得到的QpOffset有关)
    fps_factor：fps放大因子，正常情况下如果没有开启b_vfr_input这个值是恒定的
	len：一行之中宏块的数量
*/
static void mbtree_propagate_cost( int16_t *dst, uint16_t *propagate_in, uint16_t *intra_costs,
                                   uint16_t *inter_costs, uint16_t *inv_qscales, float *fps_factor, int len )
{
    float fps = *fps_factor;
    for( int i = 0; i < len; i++ ) // 遍历一行所有宏块
    {
        /* 	
        	dst[i] = X * ( intra_cost - inter_cost) / intra_cost; 
        	其中X =  propagate_in[i] + (intra_cost * inv_qscales[i] * fps)
        	
        	这里的意思其实主要就是根据比较intra_cost，inter_cost的大小来确定遗传价值
        	如果当前宏块的inter_cost大于intra_cost则认为当前宏块不需要依赖frames[p0]，所以此时整合的遗传价值就为0
        	否则inter_cost小于intra_cost则认为当前宏块的价值有一部分是来自frames[p0]的，且inter_cost越小遗传价值就会越大
        */
        int intra_cost = intra_costs[i];
        int inter_cost = X264_MIN(intra_costs[i], inter_costs[i] & LOWRES_COST_MASK);
        float propagate_intra  = intra_cost * inv_qscales[i];
        float propagate_amount = propagate_in[i] + propagate_intra*fps;
        float propagate_num    = intra_cost - inter_cost;
        float propagate_denom  = intra_cost;
        dst[i] = X264_MIN((int)(propagate_amount * propagate_num / propagate_denom + 0.5f), 32767);
    }
}
```



```c++
/*
	作用：计算frames[p0]每个宏块的遗传价值

*/
static void mbtree_propagate_list( x264_t *h, uint16_t *ref_costs, int16_t (*mvs)[2],
                                   int16_t *propagate_amount, uint16_t *lowres_costs,
                                   int bipred_weight, int mb_y, int len, int list )
{
    unsigned stride = h->mb.i_mb_stride;
    unsigned width = h->mb.i_mb_width;
    unsigned height = h->mb.i_mb_height;

    for( int i = 0; i < len; i++ )
    {
        int lists_used = lowres_costs[i]>>LOWRES_COST_SHIFT; // lists_used = 0表示lowres_costs为帧内代价

        if( !(lists_used & (1 << list)) ) // 对于non-B帧的情况(list为0), 如果是intra宏块则continue
            continue;

        int listamount = propagate_amount[i]; // propagate_amount记录了frames[b]中宏块的遗传价值
        /* Apply bipred weighting. */
        if( lists_used == 3 )
            listamount = (listamount * bipred_weight + 32) >> 6;

        /* Early termination for simple case of mv0. */
        if( !M32( mvs[i] ) )
        {
            // 如果MV为0，则将直接将遗传价值累加到参考帧相同位置宏块上
            MC_CLIP_ADD( ref_costs[mb_y*stride + i], listamount );
            continue;
        }

        int x = mvs[i][0];
        int y = mvs[i][1];
        unsigned mbx = (unsigned)((x>>5)+i);  // 当前宏块的参考宏块位置坐标 (mby,mbx)
        unsigned mby = (unsigned)((y>>5)+mb_y);
        unsigned idx0 = mbx + mby * stride;
        unsigned idx2 = idx0 + stride;
        x &= 31;
        y &= 31;
        int idx0weight = (32-y)*(32-x); // idx?weight表示运动搜索匹配的宏块在参考帧的4个宏块中的权重(重叠面积的大小)
        int idx1weight = (32-y)*x;
        int idx2weight = y*(32-x);
        int idx3weight = y*x;
        idx0weight = (idx0weight * listamount + 512) >> 10; // 按权重分配遗传价值
        idx1weight = (idx1weight * listamount + 512) >> 10;
        idx2weight = (idx2weight * listamount + 512) >> 10;
        idx3weight = (idx3weight * listamount + 512) >> 10;

        if( mbx < width-1 && mby < height-1 )
        {
            MC_CLIP_ADD( ref_costs[idx0+0], idx0weight ); // 参考帧中的4个宏块分别累加上对应的遗传价值
            MC_CLIP_ADD( ref_costs[idx0+1], idx1weight );
            MC_CLIP_ADD( ref_costs[idx2+0], idx2weight );
            MC_CLIP_ADD( ref_costs[idx2+1], idx3weight );
        }
        else
        {
            /* Note: this takes advantage of unsigned representation to
             * catch negative mbx/mby. */
            if( mby < height )
            {
                if( mbx < width )
                    MC_CLIP_ADD( ref_costs[idx0+0], idx0weight );
                if( mbx+1 < width )
                    MC_CLIP_ADD( ref_costs[idx0+1], idx1weight );
            }
            if( mby+1 < height )
            {
                if( mbx < width )
                    MC_CLIP_ADD( ref_costs[idx2+0], idx2weight );
                if( mbx+1 < width )
                    MC_CLIP_ADD( ref_costs[idx2+1], idx3weight );
            }
        }
    }
}
```





```c++
/*
	根据遗传价值确定宏块的f_qp_offset
*/
static void macroblock_tree_finish( x264_t *h, x264_frame_t *frame, float average_duration, int ref0_distance )
{
    int fps_factor = round( CLIP_DURATION(average_duration) / CLIP_DURATION(frame->f_duration) * 256 / MBTREE_PRECISION );
    float weightdelta = 0.0;
    if( ref0_distance && frame->f_weighted_cost_delta[ref0_distance-1] > 0 )
        weightdelta = (1.0 - frame->f_weighted_cost_delta[ref0_distance-1]);

    /* Allow the strength to be adjusted via qcompress, since the two
     * concepts are very similar. */
    float strength = 5.0f * (1.0f - h->param.rc.f_qcompress);
    for( int mb_index = 0; mb_index < h->mb.i_mb_count; mb_index++ )
    {
        int intra_cost = (frame->i_intra_cost[mb_index] * frame->i_inv_qscale_factor[mb_index] + 128) >> 8;
        if( intra_cost )
        {
            int propagate_cost = (frame->i_propagate_cost[mb_index] * fps_factor + 128) >> 8;
            float log2_ratio = x264_log2(intra_cost + propagate_cost) - x264_log2(intra_cost) + weightdelta;
            frame->f_qp_offset[mb_index] = frame->f_qp_offset_aq[mb_index] - strength * log2_ratio;
        }
    }
}
```

