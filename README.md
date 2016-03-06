#if Upsampling_OnTheFly
#if KW265_TILE
void upsample_ref_pic(Context* cur_slice, uint32 ref_layer_idc, int16* upsample_Y, int16* upsample_Cb, int16* upsample_Cr, int16* base_Y, int16* base_Cb, int16* base_Cr, uint8 phase_align_flag, int32 x_position, int32 y_position, int32 x_position_UV, int32 y_position_UV, int32 pu_width, int32 pu_height, int32 tile_idx)
#else
void upsample_ref_pic(Context* cur_slice, uint32 ref_layer_idc, int16* upsample_Y, int16* upsample_Cb, int16* upsample_Cr, int16* base_Y, int16* base_Cb, int16* base_Cr, uint8 phase_align_flag, int32 x_position, int32 y_position, int32 x_position_UV, int32 y_position_UV, int32 pu_width, int32 pu_height)
#endif
{
#if KW265_SIMD
	int16 i16_luma_fixed_filter[16][NTAPS_US_LUMA]=
	{
		{  0, 0,   0, 64,  0,   0,  0,  0 },
		{  0, 1,  -3, 63,  4,  -2,  1,  0 },
		{ -1, 2,  -5, 62,  8,  -3,  1,  0 },
		{ -1, 3,  -8, 60, 13,  -4,  1,  0 },
		{ -1, 4, -10, 58, 17,  -5,  1,  0 },
		{ -1, 4, -11, 52, 26,  -8,  3, -1 }, // <-> actual phase shift 1/3, used for spatial scalability x1.5
		{ -1, 3,  -9, 47, 31, -10,  4, -1 },
		{ -1, 4, -11, 45, 34, -10,  4, -1 },
		{ -1, 4, -11, 40, 40, -11,  4, -1 }, // <-> actual phase shift 1/2, equal to HEVC MC, used for spatial scalability x2
		{ -1, 4, -10, 34, 45, -11,  4, -1 },
		{ -1, 4, -10, 31, 47,  -9,  3, -1 },
		{ -1, 3,  -8, 26, 52, -11,  4, -1 }, // <-> actual phase shift 2/3, used for spatial scalability x1.5
		{  0, 1,  -5, 17, 58, -10,  4, -1 },
		{  0, 1,  -4, 13, 60,  -8,  3, -1 },
		{  0, 1,  -3,  8, 62,  -5,  2, -1 },
		{  0, 1,  -2,  4, 63,  -3,  1,  0 }
	};     ///< Luma filter taps for both 1.5x and 2x scalability

	int16 i16_chroma_fixed_filter[16][NTAPS_US_CHROMA]=
	{
		{  0, 64,  0,  0 },
		{ -2, 62,  4,  0 },
		{ -2, 58, 10, -2 },
		{ -4, 56, 14, -2 },
		{ -4, 54, 16, -2 }, // <-> actual phase shift 1/4,equal to HEVC MC, used for spatial scalability x1.5 (only for accurate Chroma alignement)
		{ -6, 52, 20, -2 }, // <-> actual phase shift 1/3, used for spatial scalability x1.5
		{ -6, 46, 28, -4 }, // <-> actual phase shift 3/8,equal to HEVC MC, used for spatial scalability x2 (only for accurate Chroma alignement)
		{ -4, 42, 30, -4 },
		{ -4, 36, 36, -4 }, // <-> actual phase shift 1/2,equal to HEVC MC, used for spatial scalability x2
		{ -4, 30, 42, -4 }, // <-> actual phase shift 7/12, used for spatial scalability x1.5 (only for accurate Chroma alignement)
		{ -4, 28, 46, -6 },
		{ -2, 20, 52, -6 }, // <-> actual phase shift 2/3, used for spatial scalability x1.5
		{ -2, 16, 54, -4 },
		{ -2, 14, 56, -4 },
		{ -2, 10, 58, -2 }, // <-> actual phase shift 7/8,equal to HEVC MC, used for spatial scalability x2 (only for accurate Chroma alignement)
		{  0,  4, 62, -2 }  // <-> actual phase shift 11/12, used for spatial scalability x1.5 (only for accurate Chroma alignement)
	}; ///< Chroma filter taps for 1.5x scalability
#else
	int32 i32_luma_fixed_filter[16][NTAPS_US_LUMA]=
	{
		{  0, 0,   0, 64,  0,   0,  0,  0 },
		{  0, 1,  -3, 63,  4,  -2,  1,  0 },
		{ -1, 2,  -5, 62,  8,  -3,  1,  0 },
		{ -1, 3,  -8, 60, 13,  -4,  1,  0 },
		{ -1, 4, -10, 58, 17,  -5,  1,  0 },
		{ -1, 4, -11, 52, 26,  -8,  3, -1 }, // <-> actual phase shift 1/3, used for spatial scalability x1.5
		{ -1, 3,  -9, 47, 31, -10,  4, -1 },
		{ -1, 4, -11, 45, 34, -10,  4, -1 },
		{ -1, 4, -11, 40, 40, -11,  4, -1 }, // <-> actual phase shift 1/2, equal to HEVC MC, used for spatial scalability x2
		{ -1, 4, -10, 34, 45, -11,  4, -1 },
		{ -1, 4, -10, 31, 47,  -9,  3, -1 },
		{ -1, 3,  -8, 26, 52, -11,  4, -1 }, // <-> actual phase shift 2/3, used for spatial scalability x1.5
		{  0, 1,  -5, 17, 58, -10,  4, -1 },
		{  0, 1,  -4, 13, 60,  -8,  3, -1 },
		{  0, 1,  -3,  8, 62,  -5,  2, -1 },
		{  0, 1,  -2,  4, 63,  -3,  1,  0 }
	};     ///< Luma filter taps for both 1.5x and 2x scalability

	int32 i32_chroma_fixed_filter[16][NTAPS_US_CHROMA]=
	{
		{  0, 64,  0,  0 },
		{ -2, 62,  4,  0 },
		{ -2, 58, 10, -2 },
		{ -4, 56, 14, -2 },
		{ -4, 54, 16, -2 }, // <-> actual phase shift 1/4,equal to HEVC MC, used for spatial scalability x1.5 (only for accurate Chroma alignement)
		{ -6, 52, 20, -2 }, // <-> actual phase shift 1/3, used for spatial scalability x1.5
		{ -6, 46, 28, -4 }, // <-> actual phase shift 3/8,equal to HEVC MC, used for spatial scalability x2 (only for accurate Chroma alignement)
		{ -4, 42, 30, -4 },
		{ -4, 36, 36, -4 }, // <-> actual phase shift 1/2,equal to HEVC MC, used for spatial scalability x2
		{ -4, 30, 42, -4 }, // <-> actual phase shift 7/12, used for spatial scalability x1.5 (only for accurate Chroma alignement)
		{ -4, 28, 46, -6 },
		{ -2, 20, 52, -6 }, // <-> actual phase shift 2/3, used for spatial scalability x1.5
		{ -2, 16, 54, -4 },
		{ -2, 14, 56, -4 },
		{ -2, 10, 58, -2 }, // <-> actual phase shift 7/8,equal to HEVC MC, used for spatial scalability x2 (only for accurate Chroma alignement)
		{  0,  4, 62, -2 }  // <-> actual phase shift 11/12, used for spatial scalability x1.5 (only for accurate Chroma alignement)
	}; ///< Chroma filter taps for 1.5x scalability
#endif
// 	int32 i32_luma_filter[16][NTAPS_US_LUMA];
// 	int32 i32_chroma_filter[16][NTAPS_US_CHROMA];

	int16 i16_luma_filter[16][NTAPS_US_LUMA];
	int16 i16_chroma_filter[16][NTAPS_US_CHROMA];

	int32 i, j;

	window sclaed_el;

	int32 i32_strideBL;

	int32  chromaFormatIdc;
	window conformance_bl;
	int32  xScal, yScal;


	int16* ui16_pSrcBufY;
	int16* ui16_pDstBufY;

	int16* i16_pSrcY;
	int16* i16_pSrcY2[4];
	int16* i16pDstY;

	int32* i16_pSrcY_1;
	int32* i16pDstY_1;

	int16* ui16_pSrcBufU;
	int16* ui16_pDstBufU;

	int16* ui16_pSrcBufV;
	int16* ui16_pDstBufV;

	int16* ui16_pSrcU;
	int16* ui16_pSrcU2[4];
	int16* ui16_pDstU;

	int16* ui16_pSrcV;
	int16* ui16_pSrcV2[4];
	int16* ui16_pDstV;

	int32 i32_scaleX;
	int32 i32_scaleY;

	int32 shift;

	uint8 b_vertPhasePositionEnableFlag;
	uint8 b_vertPhasePositionFlag;

	// O0194_JOINT_US_BITSHIFT
	uint32 currLayerId = cur_slice->slice->layerID;
	uint32 refLayerId  = cur_slice->slice->vps->refLayerId[currLayerId][ref_layer_idc];

	FILE *fp;
	FILE *fp2;
	int32 x, y;

	// O0098_SCALED_REF_LAYER_ID
	sclaed_el = get_scaled_ref_layer_window(refLayerId, cur_slice->slice->sps);

	//========== Y component upsampling ===========

	i32_scaleX = g_posScalingFactor[ref_layer_idc][0];
	i32_scaleY = g_posScalingFactor[ref_layer_idc][1];

	//mark picture is extended.
	//base_pic->is_extended=1;

	//x-j added code, the code dose not optimized yet
	ui16_pSrcBufY=base_Y;
	ui16_pSrcBufU=base_Cb;
	ui16_pSrcBufV=base_Cr;

	ui16_pDstBufY=upsample_Y;
	ui16_pDstBufU=upsample_Cb;
	ui16_pDstBufV=upsample_Cr;

	// Q0200_CONFORMANCE_BL_SIZE
	chromaFormatIdc = cur_slice->slice->pcBaseColPic[ref_layer_idc]->slices[0]->vps->vpsRepFormat->chromaFormatVpsIdc;
	conformance_bl  = cur_slice->slice->pcBaseColPic[ref_layer_idc]->slices[0]->sps->conformance_window;
	xScal = winUnitX[chromaFormatIdc];
	yScal = winUnitY[chromaFormatIdc];

	// P0312_VERT_PHASE_ADJ
	b_vertPhasePositionEnableFlag = sclaed_el.vert_phase_position_enable_flag;
	b_vertPhasePositionFlag       = cur_slice->slice->vertPhasePositionFlag[ref_layer_idc];

	if( b_vertPhasePositionFlag )
	{
		assert( b_vertPhasePositionEnableFlag );
	}

	assert ( NTAPS_US_LUMA == 8 );
	assert ( NTAPS_US_CHROMA == 4 );
#if KW265_SIMD
	{
		//upsampling
		int32 refPos16 = 0;
		int32 refPos16_1[4] = {0};
		int32 phase    = 0;
		int32 refPos   = 0;
		int16* coeff = i16_chroma_filter [ phase ];
		int16* coeff16;

		int32 refPos16_y = 0;
		int32 refPos_y = 0;
		int32 refPos16_uv = 0;
		int32 refPos_uv =0;

		int32   shiftX = 16;
		int32   shiftY = 16;

		int32 shiftXM4 ;
		int32 shiftYM4 ;

		int32 leftStartL  ;
		int32 rightEndL   ;
		int32 topStartL   ;
		int32 bottomEndL  ;
		int32 leftOffset  ;

		int32 shift1      ;

		int32 nShift;
		int32 iOffset;

		int32 leftStartC ;
		int32 rightEndC  ;
		int32 topStartC  ;
		int32 bottomEndC ;

		int32 phaseXC;
		int32 phaseYC;

		//for Luma, if Phase 0, then both PhaseX  and PhaseY should be 0. If symmetric: both PhaseX and PhaseY should be 2
		int32   phaseX = 2 * phase_align_flag;
		int32   phaseY = 2 * phase_align_flag;

		int32   addX = ( ( phaseX * i32_scaleX + 2 ) >> 2 ) + ( 1 << ( shiftX - 5 ) );
		int32   addY = ( ( phaseY * i32_scaleY + 2 ) >> 2 ) + ( 1 << ( shiftY - 5 ) );

		int32   deltaX = (int32)phase_align_flag <<3;
		int32   deltaY = (((int32)phase_align_flag <<3)>>(int32)b_vertPhasePositionEnableFlag) + ((int32)b_vertPhasePositionFlag<<3);

		int32 n=0;
		int32 n1=0;

		__m128i xmm_coeff[4];
		__m128i xmm_i16_pSrcY[8];
		__m128i xmm_ui16_pSrcU[8];
		__m128i xmm_ui16_pSrcV[8];

		__m128i r1,r2,r3,r4,r5,r6,r7,r8;
		__m128i xmm_zero  = _mm_setzero_si128();
		__m128i xmax = _mm_set1_epi16(MAX_PIX);
		__m128i xmm_iOffset;

		__m128i xmm_coeff0;
		__m128i xmm_coeff1;
		__m128i xmm_coeff2;
		__m128i xmm_coeff3;
		__m128i xmm_coeff4;
		__m128i xmm_coeff5;
		__m128i xmm_coeff6;
		__m128i xmm_coeff7;

		FILE *fp;
		int32 ii,jj,kk;
		char filename[1000];

// 		for ( i = 0; i < 16; i++)
// 		{
// 			memcpy( i16_luma_filter[i],   i16_luma_fixed_filter[i], sizeof(int16) * NTAPS_US_LUMA   );
// 			memcpy( i16_chroma_filter[i], i16_chroma_fixed_filter[i], sizeof(int16) * NTAPS_US_CHROMA );
// 		}

		i32_strideBL  = Base_Y_stride;

		deltaX -= ( conformance_bl.left_crop_offset * xScal ) << 4;
		deltaY -= ( conformance_bl.top_crop_offset * yScal  ) << 4;

		shiftXM4 = shiftX - 4;
		shiftYM4 = shiftY - 4;

		leftStartL = sclaed_el.left_crop_offset;
		rightEndL  = cur_slice->pic_size->width;
		topStartL  = sclaed_el.top_crop_offset;
		bottomEndL = cur_slice->pic_size->height;
		leftOffset = leftStartL > 0 ? leftStartL : 0;

		// g_bitDepthY was set to EL bit-depth, but shift1 should be calculated using BL bit-depth
		shift1 = 0; // g_bitDepthYLayer[refLayerId] - 8;

		if( cur_slice->slice->pps->nCGSFlag )
		{
			shift1 = cur_slice->slice->pps->nCGSOutputBitDepthY - 8;
		}


		//========== horizontal upsampling ===========




		for( i = x_position*2; i < (x_position*2)+pu_width; i += 4 ) ////////////////////////////////////////////////////////////////////////////////////////////
		{
			////1
			int32 x = Clip3( int32, leftStartL, rightEndL - 1, i );
			int32 y = Clip3(int32, topStartL, bottomEndL - 1, y_position*2);

			refPos16_1[0] = (((x - leftStartL)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[0] & 15;
			refPos   = refPos16_1[0] >> 4;
			coeff = i16_luma_fixed_filter[phase];
			xmm_coeff[0] = _mm_loadu_si128((__m128i *) coeff);

			refPos16_y = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_y   = refPos16_y >> 4;

			i16_pSrcY2[0] = ui16_pSrcBufY + refPos -((NTAPS_US_LUMA>>1) - 1);
			i16_pSrcY2[0] += ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*(i32_strideBL));

			////2
			x = Clip3( int32, leftStartL, rightEndL - 1, i+1 );
			y = Clip3(int32, topStartL, bottomEndL - 1, y_position*2);

			refPos16_1[1] = (((x - leftStartL)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[1] & 15;
			refPos   = refPos16_1[1] >> 4;
			coeff = i16_luma_fixed_filter[phase];
			xmm_coeff[1] = _mm_loadu_si128((__m128i *) coeff);

			refPos16_y = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_y   = refPos16_y >> 4;

			i16_pSrcY2[1] = ui16_pSrcBufY + refPos -((NTAPS_US_LUMA>>1) - 1);
			i16_pSrcY2[1] += ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*(i32_strideBL));

			////3
			x = Clip3( int32, leftStartL, rightEndL - 1, i+2 );
			y = Clip3(int32, topStartL, bottomEndL - 1, y_position*2);

			refPos16_1[2] = (((x - leftStartL)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[2] & 15;
			refPos   = refPos16_1[2] >> 4;
			coeff = i16_luma_fixed_filter[phase];
			xmm_coeff[2] = _mm_loadu_si128((__m128i *) coeff);

			refPos16_y = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_y   = refPos16_y >> 4;

			i16_pSrcY2[2] = ui16_pSrcBufY + refPos -((NTAPS_US_LUMA>>1) - 1);
			i16_pSrcY2[2] += ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*(i32_strideBL));

			////4
			x = Clip3( int32, leftStartL, rightEndL - 1, i+3 );
			y = Clip3(int32, topStartL, bottomEndL - 1, y_position*2);

			refPos16_1[3] = (((x - leftStartL)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[3] & 15;
			refPos   = refPos16_1[3] >> 4;
			coeff = i16_luma_fixed_filter[phase];
			xmm_coeff[3] = _mm_loadu_si128((__m128i *) coeff);

			refPos16_y = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_y   = refPos16_y >> 4;

			i16_pSrcY2[3] = ui16_pSrcBufY + refPos -((NTAPS_US_LUMA>>1) - 1);
			i16_pSrcY2[3] += ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*(i32_strideBL));

#if KW265_TILE
			i16pDstY = ui16_pTempBufY[tile_idx] + n;   
#else
			i16pDstY = ui16_pTempBufY + n;
#endif

			for( j = 0; j < (pu_height/2)+7 ; j++ )
			{
				xmm_i16_pSrcY[0] = _mm_loadu_si128((__m128i *) i16_pSrcY2[0]);
				xmm_i16_pSrcY[1] = _mm_loadu_si128((__m128i *) i16_pSrcY2[1]);
				xmm_i16_pSrcY[2] = _mm_loadu_si128((__m128i *) i16_pSrcY2[2]);
				xmm_i16_pSrcY[3] = _mm_loadu_si128((__m128i *) i16_pSrcY2[3]);

				r1 = _mm_madd_epi16(xmm_i16_pSrcY[0], xmm_coeff[0]);
				r2 = _mm_madd_epi16(xmm_i16_pSrcY[1], xmm_coeff[1]);
				r3 = _mm_madd_epi16(xmm_i16_pSrcY[2], xmm_coeff[2]);
				r4 = _mm_madd_epi16(xmm_i16_pSrcY[3], xmm_coeff[3]);

				r1 = _mm_hadd_epi32(r1,r2);
				r3 = _mm_hadd_epi32(r3,r4);

				r1 = _mm_hadd_epi32(r1,r3);

				//32-bit to 16-bit
				r1 = _mm_packs_epi32(r1,xmm_zero);

				_mm_storel_epi64((__m128i *)i16pDstY, r1);

				i16_pSrcY2[0] += i32_strideBL;
				i16_pSrcY2[1] += i32_strideBL;
				i16_pSrcY2[2] += i32_strideBL;
				i16_pSrcY2[3] += i32_strideBL;

				i16pDstY += pu_width;

			}
			n += 4;
		}

		//========== vertical upsampling ===========
		nShift = 12; // 20 - g_bitDepthYLayer[currLayerId];
		iOffset = 1 << (nShift - 1);
		xmm_iOffset = _mm_set1_epi32(iOffset);

		for( j = y_position*2; j < (y_position*2)+pu_height; j++ )
		{
			int16* i16_pDstY0;
			int32 y = Clip3(int32, topStartL, bottomEndL - 1, j);

			refPos16 = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			phase    = refPos16 & 15;
			refPos   = refPos16 >> 4;
			coeff = i16_luma_fixed_filter[phase];

#if KW265_TILE
			i16_pSrcY = ui16_pTempBufY[tile_idx] + (refPos -((NTAPS_US_LUMA>>1) - 1))*pu_width - ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*pu_width);  //refPos -((NTAPS_US_LUMA>>1) - 1))*i32_strideEL
#else
			i16_pSrcY = ui16_pTempBufY + (refPos -((NTAPS_US_LUMA>>1) - 1))*pu_width - ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*pu_width);  //refPos -((NTAPS_US_LUMA>>1) - 1))*i32_strideEL
#endif
			i16_pDstY0 = ui16_pDstBufY + (j-(y_position*2)) * pu_width;

			i16pDstY = i16_pDstY0 + leftOffset;
			i16_pSrcY += leftOffset;

			for( i = pu_width; i > 0; i -= 4 )
			{
				xmm_i16_pSrcY[0] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 0*pu_width));
				xmm_i16_pSrcY[1] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 1*pu_width));
				xmm_i16_pSrcY[2] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 2*pu_width));
				xmm_i16_pSrcY[3] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 3*pu_width));
				xmm_i16_pSrcY[4] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 4*pu_width));
				xmm_i16_pSrcY[5] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 5*pu_width));
				xmm_i16_pSrcY[6] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 6*pu_width));
				xmm_i16_pSrcY[7] = _mm_loadu_si128((__m128i *) (i16_pSrcY + 7*pu_width));

				xmm_coeff0 = _mm_set1_epi32(coeff[0]);
				xmm_coeff1 = _mm_set1_epi32(coeff[1]);
				xmm_coeff2 = _mm_set1_epi32(coeff[2]);
				xmm_coeff3 = _mm_set1_epi32(coeff[3]);
				xmm_coeff4 = _mm_set1_epi32(coeff[4]);
				xmm_coeff5 = _mm_set1_epi32(coeff[5]);
				xmm_coeff6 = _mm_set1_epi32(coeff[6]);
				xmm_coeff7 = _mm_set1_epi32(coeff[7]);

				r1 = _mm_unpacklo_epi16(xmm_i16_pSrcY[0], _mm_cmplt_epi16(xmm_i16_pSrcY[0], xmm_zero));
				r1 = _mm_mullo_epi32(r1, xmm_coeff0);

				r2 = _mm_unpacklo_epi16(xmm_i16_pSrcY[1], _mm_cmplt_epi16(xmm_i16_pSrcY[1], xmm_zero));
				r2 = _mm_mullo_epi32(r2, xmm_coeff1);

				r3 = _mm_unpacklo_epi16(xmm_i16_pSrcY[2], _mm_cmplt_epi16(xmm_i16_pSrcY[2], xmm_zero));
				r3 = _mm_mullo_epi32(r3, xmm_coeff2);

				r4 = _mm_unpacklo_epi16(xmm_i16_pSrcY[3], _mm_cmplt_epi16(xmm_i16_pSrcY[3], xmm_zero));
				r4 = _mm_mullo_epi32(r4, xmm_coeff3);

				r5 = _mm_unpacklo_epi16(xmm_i16_pSrcY[4], _mm_cmplt_epi16(xmm_i16_pSrcY[4], xmm_zero));
				r5 = _mm_mullo_epi32(r5, xmm_coeff4);

				r6 = _mm_unpacklo_epi16(xmm_i16_pSrcY[5], _mm_cmplt_epi16(xmm_i16_pSrcY[5], xmm_zero));
				r6 = _mm_mullo_epi32(r6, xmm_coeff5);

				r7 = _mm_unpacklo_epi16(xmm_i16_pSrcY[6], _mm_cmplt_epi16(xmm_i16_pSrcY[6], xmm_zero));
				r7 = _mm_mullo_epi32(r7, xmm_coeff6);

				r8 = _mm_unpacklo_epi16(xmm_i16_pSrcY[7], _mm_cmplt_epi16(xmm_i16_pSrcY[7], xmm_zero));
				r8 = _mm_mullo_epi32(r8, xmm_coeff7);

				r1 = _mm_add_epi32(r1,r2);
				r1 = _mm_add_epi32(r1,r3);
				r1 = _mm_add_epi32(r1,r4);
				r1 = _mm_add_epi32(r1,r5);
				r1 = _mm_add_epi32(r1,r6);
				r1 = _mm_add_epi32(r1,r7);
				r1 = _mm_add_epi32(r1,r8);

				//scale down
				r1 = _mm_add_epi32(r1, xmm_iOffset);
				r1  = _mm_srai_epi32( r1, nShift);

				//32bit to 16bit
				r1 = _mm_packs_epi32(r1,r1);

				//clipping
				r1	= _mm_max_epi16(r1, xmm_zero);
				r1	= _mm_min_epi16(r1, xmax);

				_mm_storel_epi64((__m128i *)i16pDstY,r1);

				i16_pSrcY += 4;
				i16pDstY += 4;
			}
		}

		// 	i32_widthBL  = base_pic->size->width;
		// 	i32_heightBL = base_pic->size->height;

		//========== UV component upsampling ===========

		i32_strideBL  = Base_UV_stride;

		leftStartC = sclaed_el.left_crop_offset >> 1;
		rightEndC  = cur_slice->pic_size->width >> 1;
		topStartC  = sclaed_el.top_crop_offset;
		bottomEndC = cur_slice->pic_size->height >> 1;
		leftOffset = leftStartC > 0 ? leftStartC : 0;
		shiftX = 16;
		shiftY = 16;

		phaseXC = phase_align_flag;
		phaseYC = phase_align_flag + 1;

		addX       = ( ( phaseXC * i32_scaleX + 2 ) >> 2 ) + ( 1 << ( shiftX - 5 ) );
		addY       = ( ( phaseYC * i32_scaleY + 2 ) >> 2 ) + ( 1 << ( shiftY - 5 ) );

		deltaX     = (int32)phase_align_flag << 2;
		deltaY     = ((( (int32)phase_align_flag +1)<<2)>>(int32)b_vertPhasePositionEnableFlag)+((int32)b_vertPhasePositionFlag<<3);

		deltaX  -= ( ( conformance_bl.left_crop_offset * xScal ) >> 1 ) << 4;
		deltaY  -= ( ( conformance_bl.top_crop_offset  * yScal ) >> 1 ) << 4;

		shiftXM4 = shiftX - 4;
		shiftYM4 = shiftY - 4;

		// g_bitDepthC was set to EL bit-depth, but shift1 should be calculated using BL bit-depth
		shift1 = 0; // g_bitDepthCLayer[refLayerId] - 8;

		if( cur_slice->slice->pps->nCGSFlag )
		{
			shift1 = cur_slice->slice->pps->nCGSOutputBitDepthC- 8;
		}

		//========== horizontal upsampling ===========
		if((pu_width>>1) == 2)
		{
			for( i = x_position_UV*2; i < (x_position_UV*2)+(pu_width>>1); i++ )
			{
				int32 x = Clip3(int32, leftStartC, rightEndC - 1, i);
				int32 y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

				refPos16 = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
				phase    = refPos16 & 15;
				refPos   = refPos16 >> 4;
				coeff = i16_chroma_fixed_filter[phase];

				refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				refPos_uv   = refPos16_uv >> 4;

				ui16_pSrcU = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcU += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +
				ui16_pSrcV = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcV += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));
#if KW265_TILE
				ui16_pDstU = ui16_pTempBufU[tile_idx] +n1;
				ui16_pDstV = ui16_pTempBufV[tile_idx] +n1;
#else
				ui16_pDstU = ui16_pTempBufU +n1;
				ui16_pDstV = ui16_pTempBufV +n1;
#endif

				for( j = 0; j < ((pu_height>>1)/2)+4 ; j++ )
				{
					int32 sum_chroma_hor_u = ui16_pSrcU[0]*coeff[0] + ui16_pSrcU[1]*coeff[1] + ui16_pSrcU[2]*coeff[2] + ui16_pSrcU[3]*coeff[3];
					int32 sum_chroma_hor_v = ui16_pSrcV[0]*coeff[0] + ui16_pSrcV[1]*coeff[1] + ui16_pSrcV[2]*coeff[2] + ui16_pSrcV[3]*coeff[3];

					*ui16_pDstU = sum_chroma_hor_u >> shift1;
					*ui16_pDstV = sum_chroma_hor_v >> shift1;

					ui16_pSrcU += i32_strideBL;
					ui16_pSrcV += i32_strideBL;
					ui16_pDstU += pu_width>>1;
					ui16_pDstV += pu_width>>1;
				}
				n1++;
			}
		}
		else if((pu_width>>1) == 6)
		{
			i = x_position_UV*2;
			//////1
			x = Clip3(int32, leftStartC, rightEndC - 1, i);
			y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

			refPos16_1[0] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[0] & 15;
			refPos   = refPos16_1[0] >> 4;
			coeff = i16_chroma_fixed_filter[phase];
			//xmm_coeff[0] = _mm_loadu_si128((__m128i *) coeff);
			xmm_coeff[0] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);

			refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_uv   = refPos16_uv >> 4;

			ui16_pSrcU2[0] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcU2[0] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

			ui16_pSrcV2[0] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcV2[0] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

			//////2
			x = Clip3(int32, leftStartC, rightEndC - 1, i+1);
			y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

			refPos16_1[1] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[1] & 15;
			refPos   = refPos16_1[1] >> 4;
			coeff = i16_chroma_fixed_filter[phase];
			//xmm_coeff[1] = _mm_loadu_si128((__m128i *) coeff);
			xmm_coeff[1] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);


			refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_uv   = refPos16_uv >> 4;

			ui16_pSrcU2[1] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcU2[1] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

			ui16_pSrcV2[1] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcV2[1] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

			//////3
			x = Clip3(int32, leftStartC, rightEndC - 1, i+2);
			y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

			refPos16_1[2] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[2] & 15;
			refPos   = refPos16_1[2] >> 4;
			coeff = i16_chroma_fixed_filter[phase];
			//xmm_coeff[2] = _mm_loadu_si128((__m128i *) coeff);
			xmm_coeff[2] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);

			refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_uv   = refPos16_uv >> 4;

			ui16_pSrcU2[2] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcU2[2] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

			ui16_pSrcV2[2] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcV2[2] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

			//////4
			x = Clip3(int32, leftStartC, rightEndC - 1, i+3);
			y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

			refPos16_1[3] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16_1[3] & 15;
			refPos   = refPos16_1[3] >> 4;
			coeff = i16_chroma_fixed_filter[phase];
			//xmm_coeff[3] = _mm_loadu_si128((__m128i *) coeff);
			xmm_coeff[3] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);

			refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_uv   = refPos16_uv >> 4;

			ui16_pSrcU2[3] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcU2[3] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

			ui16_pSrcV2[3] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcV2[3] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

#if KW265_TILE
				ui16_pDstU = ui16_pTempBufU[tile_idx];
				ui16_pDstV = ui16_pTempBufV[tile_idx];
#else
				ui16_pDstU = ui16_pTempBufU;
				ui16_pDstV = ui16_pTempBufV;
#endif

				for( j = 0; j < ((pu_height>>1)/2)+4 ; j++ )
				{
					xmm_ui16_pSrcU[0] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[0]);
					xmm_ui16_pSrcU[1] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[1]);
					xmm_ui16_pSrcU[2] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[2]);
					xmm_ui16_pSrcU[3] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[3]);

					r1 = _mm_madd_epi16(xmm_ui16_pSrcU[0], xmm_coeff[0]);
					r2 = _mm_madd_epi16(xmm_ui16_pSrcU[1], xmm_coeff[1]);
					r3 = _mm_madd_epi16(xmm_ui16_pSrcU[2], xmm_coeff[2]);
					r4 = _mm_madd_epi16(xmm_ui16_pSrcU[3], xmm_coeff[3]);

					r1 = _mm_hadd_epi32(r1,r2);
					r3 = _mm_hadd_epi32(r3,r4);

					r1 = _mm_hadd_epi32(r1,r3);

					//32-bit to 16-bit
					r1 = _mm_packs_epi32(r1,xmm_zero);

					r1  = _mm_srai_epi32( r1, shift1);

					_mm_storel_epi64((__m128i *) ui16_pDstU, r1);

					xmm_ui16_pSrcV[0] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[0]);
					xmm_ui16_pSrcV[1] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[1]);
					xmm_ui16_pSrcV[2] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[2]);
					xmm_ui16_pSrcV[3] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[3]);

					r1 = _mm_madd_epi16(xmm_ui16_pSrcV[0], xmm_coeff[0]);
					r2 = _mm_madd_epi16(xmm_ui16_pSrcV[1], xmm_coeff[1]);
					r3 = _mm_madd_epi16(xmm_ui16_pSrcV[2], xmm_coeff[2]);
					r4 = _mm_madd_epi16(xmm_ui16_pSrcV[3], xmm_coeff[3]);

					r1 = _mm_hadd_epi32(r1,r2);
					r3 = _mm_hadd_epi32(r3,r4);

					r1 = _mm_hadd_epi32(r1,r3);

					//32-bit to 16-bit

					r1  = _mm_srai_epi32( r1, shift1);

					r1 = _mm_packs_epi32(r1,xmm_zero);


					_mm_storel_epi64((__m128i *) ui16_pDstV, r1);

					ui16_pSrcU2[0] += i32_strideBL;
					ui16_pSrcU2[1] += i32_strideBL;
					ui16_pSrcU2[2] += i32_strideBL;
					ui16_pSrcU2[3] += i32_strideBL;

					ui16_pSrcV2[0] += i32_strideBL;
					ui16_pSrcV2[1] += i32_strideBL;
					ui16_pSrcV2[2] += i32_strideBL;
					ui16_pSrcV2[3] += i32_strideBL;

					ui16_pDstU += pu_width>>1;
					ui16_pDstV += pu_width>>1;
				}

				for( i = x_position_UV*2 + 4; i < (x_position_UV*2)+(pu_width>>1); i++ )
				{
					x = Clip3(int32, leftStartC, rightEndC - 1, i);
					y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

					refPos16 = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
					phase    = refPos16 & 15;
					refPos   = refPos16 >> 4;
					coeff = i16_chroma_fixed_filter[phase];

					refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
					refPos_uv   = refPos16_uv >> 4;

					ui16_pSrcU = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
					ui16_pSrcU += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +
					ui16_pSrcV = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
					ui16_pSrcV += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));
#if KW265_TILE
					ui16_pDstU = ui16_pTempBufU[tile_idx] + 4 + n1;
					ui16_pDstV = ui16_pTempBufV[tile_idx] + 4 + n1;
#else
					ui16_pDstU = ui16_pTempBufU + 4 + n1;
					ui16_pDstV = ui16_pTempBufV + 4 + n1;
#endif

					for( j = 0; j < ((pu_height>>1)/2)+4 ; j++ )
					{
						int32 sum_chroma_hor_u = ui16_pSrcU[0]*coeff[0] + ui16_pSrcU[1]*coeff[1] + ui16_pSrcU[2]*coeff[2] + ui16_pSrcU[3]*coeff[3];
						int32 sum_chroma_hor_v = ui16_pSrcV[0]*coeff[0] + ui16_pSrcV[1]*coeff[1] + ui16_pSrcV[2]*coeff[2] + ui16_pSrcV[3]*coeff[3];

						*ui16_pDstU = sum_chroma_hor_u >> shift1;
						*ui16_pDstV = sum_chroma_hor_v >> shift1;

						ui16_pSrcU += i32_strideBL;
						ui16_pSrcV += i32_strideBL;
						ui16_pDstU += pu_width>>1;
						ui16_pDstV += pu_width>>1;
					}
					n1++;
				}
			
		}
		else
		{
			for( i = x_position_UV*2; i < (x_position_UV*2)+(pu_width>>1); i += 4 )
			{
				//////1
				int32 x = Clip3(int32, leftStartC, rightEndC - 1, i);
				int32 y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

				refPos16_1[0] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
				phase    = refPos16_1[0] & 15;
				refPos   = refPos16_1[0] >> 4;
				coeff = i16_chroma_fixed_filter[phase];
				//xmm_coeff[0] = _mm_loadu_si128((__m128i *) coeff);
				xmm_coeff[0] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);

				refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				refPos_uv   = refPos16_uv >> 4;

				ui16_pSrcU2[0] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcU2[0] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

				ui16_pSrcV2[0] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcV2[0] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

				//////2
				x = Clip3(int32, leftStartC, rightEndC - 1, i+1);
				y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

				refPos16_1[1] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
				phase    = refPos16_1[1] & 15;
				refPos   = refPos16_1[1] >> 4;
				coeff = i16_chroma_fixed_filter[phase];
				//xmm_coeff[1] = _mm_loadu_si128((__m128i *) coeff);
				xmm_coeff[1] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);


				refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				refPos_uv   = refPos16_uv >> 4;

				ui16_pSrcU2[1] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcU2[1] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

				ui16_pSrcV2[1] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcV2[1] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

				//////3
				x = Clip3(int32, leftStartC, rightEndC - 1, i+2);
				y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

				refPos16_1[2] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
				phase    = refPos16_1[2] & 15;
				refPos   = refPos16_1[2] >> 4;
				coeff = i16_chroma_fixed_filter[phase];
				//xmm_coeff[2] = _mm_loadu_si128((__m128i *) coeff);
				xmm_coeff[2] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);

				refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				refPos_uv   = refPos16_uv >> 4;

				ui16_pSrcU2[2] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcU2[2] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

				ui16_pSrcV2[2] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcV2[2] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

				//////4
				x = Clip3(int32, leftStartC, rightEndC - 1, i+3);
				y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

				refPos16_1[3] = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
				phase    = refPos16_1[3] & 15;
				refPos   = refPos16_1[3] >> 4;
				coeff = i16_chroma_fixed_filter[phase];
				//xmm_coeff[3] = _mm_loadu_si128((__m128i *) coeff);
				xmm_coeff[3] = _mm_set_epi16(0,0,0,0,coeff[3],coeff[2],coeff[1],coeff[0]);

				refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				refPos_uv   = refPos16_uv >> 4;

				ui16_pSrcU2[3] = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcU2[3] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +

				ui16_pSrcV2[3] = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
				ui16_pSrcV2[3] += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));

#if KW265_TILE
				ui16_pDstU = ui16_pTempBufU[tile_idx] +n1;
				ui16_pDstV = ui16_pTempBufV[tile_idx] +n1;
#else
				ui16_pDstU = ui16_pTempBufU +n1;
				ui16_pDstV = ui16_pTempBufV +n1;
#endif

				for( j = 0; j < ((pu_height>>1)/2)+4 ; j++ )
				{
					xmm_ui16_pSrcU[0] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[0]);
					xmm_ui16_pSrcU[1] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[1]);
					xmm_ui16_pSrcU[2] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[2]);
					xmm_ui16_pSrcU[3] = _mm_loadu_si128((__m128i *) ui16_pSrcU2[3]);

					r1 = _mm_madd_epi16(xmm_ui16_pSrcU[0], xmm_coeff[0]);
					r2 = _mm_madd_epi16(xmm_ui16_pSrcU[1], xmm_coeff[1]);
					r3 = _mm_madd_epi16(xmm_ui16_pSrcU[2], xmm_coeff[2]);
					r4 = _mm_madd_epi16(xmm_ui16_pSrcU[3], xmm_coeff[3]);

					r1 = _mm_hadd_epi32(r1,r2);
					r3 = _mm_hadd_epi32(r3,r4);

					r1 = _mm_hadd_epi32(r1,r3);

					//32-bit to 16-bit
					r1 = _mm_packs_epi32(r1,xmm_zero);

					r1  = _mm_srai_epi32( r1, shift1);

					_mm_storel_epi64((__m128i *) ui16_pDstU, r1);

					xmm_ui16_pSrcV[0] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[0]);
					xmm_ui16_pSrcV[1] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[1]);
					xmm_ui16_pSrcV[2] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[2]);
					xmm_ui16_pSrcV[3] = _mm_loadu_si128((__m128i *) ui16_pSrcV2[3]);

					r1 = _mm_madd_epi16(xmm_ui16_pSrcV[0], xmm_coeff[0]);
					r2 = _mm_madd_epi16(xmm_ui16_pSrcV[1], xmm_coeff[1]);
					r3 = _mm_madd_epi16(xmm_ui16_pSrcV[2], xmm_coeff[2]);
					r4 = _mm_madd_epi16(xmm_ui16_pSrcV[3], xmm_coeff[3]);

					r1 = _mm_hadd_epi32(r1,r2);
					r3 = _mm_hadd_epi32(r3,r4);

					r1 = _mm_hadd_epi32(r1,r3);

					//32-bit to 16-bit

					r1  = _mm_srai_epi32( r1, shift1);

					r1 = _mm_packs_epi32(r1,xmm_zero);


					_mm_storel_epi64((__m128i *) ui16_pDstV, r1);

					ui16_pSrcU2[0] += i32_strideBL;
					ui16_pSrcU2[1] += i32_strideBL;
					ui16_pSrcU2[2] += i32_strideBL;
					ui16_pSrcU2[3] += i32_strideBL;

					ui16_pSrcV2[0] += i32_strideBL;
					ui16_pSrcV2[1] += i32_strideBL;
					ui16_pSrcV2[2] += i32_strideBL;
					ui16_pSrcV2[3] += i32_strideBL;

					ui16_pDstU += pu_width>>1;
					ui16_pDstV += pu_width>>1;
				}
				n1 += 4;
			}

		}
		//========== vertical upsampling ===========

		nShift = 12; // 20 - g_bitDepthCLayer[currLayerId];

		iOffset = 1 << (nShift - 1);

		if((pu_width>>1) == 2)
		{
			for( j = y_position_UV*2; j < (y_position_UV*2)+(pu_height>>1); j++ )
			{
				int16* ui8_pDstU0;
				int16* ui8_pDstV0;
				int32 y = Clip3(int32, topStartC, bottomEndC - 1, j);
				refPos16 = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				phase    = refPos16 & 15;
				refPos   = refPos16 >> 4;
				coeff = i16_chroma_fixed_filter[phase];

#if KW265_TILE
				ui16_pSrcU = ui16_pTempBufU[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
				ui16_pSrcV = ui16_pTempBufV[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#else
				ui16_pSrcU = ui16_pTempBufU  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
				ui16_pSrcV = ui16_pTempBufV  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#endif



				ui8_pDstU0 = ui16_pDstBufU + (j-(y_position_UV*2))*(pu_width>>1);
				ui8_pDstV0 = ui16_pDstBufV + (j-(y_position_UV*2))*(pu_width>>1);

				ui16_pDstU = ui8_pDstU0 + leftOffset;
				ui16_pDstV = ui8_pDstV0 + leftOffset;
				ui16_pSrcU += leftOffset;
				ui16_pSrcV += leftOffset;

				for( i = pu_width>>1 ; i > 0; i-- )
				{
					int32 sum_chroma_ver_u = ui16_pSrcU[0]*coeff[0] + ui16_pSrcU[pu_width>>1]*coeff[1] + ui16_pSrcU[2*(pu_width>>1)]*coeff[2] + ui16_pSrcU[3*(pu_width>>1)]*coeff[3];
					int32 sum_chroma_ver_v = ui16_pSrcV[0]*coeff[0] + ui16_pSrcV[pu_width>>1]*coeff[1] + ui16_pSrcV[2*(pu_width>>1)]*coeff[2] + ui16_pSrcV[3*(pu_width>>1)]*coeff[3];

					*ui16_pDstU = Clip3(int32, 0, ((1<<8)-1), (sum_chroma_ver_u + iOffset) >> (nShift));       
					*ui16_pDstV = Clip3(int32, 0, ((1<<8)-1), (sum_chroma_ver_v + iOffset) >> (nShift));

					ui16_pSrcU++;
					ui16_pSrcV++;
					ui16_pDstU++;
					ui16_pDstV++;
				}

			}
		}
		else if((pu_width>>1) == 6)
		{


			for( j = y_position_UV*2; j < (y_position_UV*2)+(pu_height>>1); j++ )
			{
				int16* ui8_pDstU0;
				int16* ui8_pDstV0;
				int32 y = Clip3(int32, topStartC, bottomEndC - 1, j);
				refPos16 = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				phase    = refPos16 & 15;
				refPos   = refPos16 >> 4;
				coeff = i16_chroma_fixed_filter[phase];

#if KW265_TILE
				ui16_pSrcU = ui16_pTempBufU[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
				ui16_pSrcV = ui16_pTempBufV[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#else
				ui16_pSrcU = ui16_pTempBufU  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
				ui16_pSrcV = ui16_pTempBufV  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#endif

				ui8_pDstU0 = ui16_pDstBufU + (j-(y_position_UV*2))*(pu_width>>1);
				ui8_pDstV0 = ui16_pDstBufV + (j-(y_position_UV*2))*(pu_width>>1);

				ui16_pDstU = ui8_pDstU0 + leftOffset;
				ui16_pDstV = ui8_pDstV0 + leftOffset;
				ui16_pSrcU += leftOffset;
				ui16_pSrcV += leftOffset;


				xmm_ui16_pSrcU[0] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 0*(pu_width>>1)));
				xmm_ui16_pSrcU[1] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 1*(pu_width>>1)));
				xmm_ui16_pSrcU[2] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 2*(pu_width>>1)));
				xmm_ui16_pSrcU[3] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 3*(pu_width>>1)));

				xmm_coeff0 = _mm_set1_epi32(coeff[0]);
				xmm_coeff1 = _mm_set1_epi32(coeff[1]);
				xmm_coeff2 = _mm_set1_epi32(coeff[2]);
				xmm_coeff3 = _mm_set1_epi32(coeff[3]);

				r1 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[0], _mm_cmplt_epi16(xmm_ui16_pSrcU[0], xmm_zero));
				r1 = _mm_mullo_epi32(r1, xmm_coeff0);

				r2 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[1], _mm_cmplt_epi16(xmm_ui16_pSrcU[1], xmm_zero));
				r2 = _mm_mullo_epi32(r2, xmm_coeff1);

				r3 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[2], _mm_cmplt_epi16(xmm_ui16_pSrcU[2], xmm_zero));
				r3 = _mm_mullo_epi32(r3, xmm_coeff2);

				r4 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[3], _mm_cmplt_epi16(xmm_ui16_pSrcU[3], xmm_zero));
				r4 = _mm_mullo_epi32(r4, xmm_coeff3);

				r1 = _mm_add_epi32(r1,r2);
				r1 = _mm_add_epi32(r1,r3);
				r1 = _mm_add_epi32(r1,r4);

				//scale down
				r1 = _mm_add_epi32(r1, xmm_iOffset);
				r1  = _mm_srai_epi32( r1, nShift);

				//32bit to 16bit
				r1 = _mm_packs_epi32(r1,r1);

				//clipping
				r1	= _mm_max_epi16(r1, xmm_zero);
				r1	= _mm_min_epi16(r1, xmax);

				_mm_storel_epi64((__m128i *)ui16_pDstU,r1);

				xmm_ui16_pSrcV[0] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 0*(pu_width>>1)));
				xmm_ui16_pSrcV[1] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 1*(pu_width>>1)));
				xmm_ui16_pSrcV[2] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 2*(pu_width>>1)));
				xmm_ui16_pSrcV[3] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 3*(pu_width>>1)));


				r1 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[0], _mm_cmplt_epi16(xmm_ui16_pSrcV[0], xmm_zero));
				r1 = _mm_mullo_epi32(r1, xmm_coeff0);

				r2 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[1], _mm_cmplt_epi16(xmm_ui16_pSrcV[1], xmm_zero));
				r2 = _mm_mullo_epi32(r2, xmm_coeff1);

				r3 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[2], _mm_cmplt_epi16(xmm_ui16_pSrcV[2], xmm_zero));
				r3 = _mm_mullo_epi32(r3, xmm_coeff2);

				r4 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[3], _mm_cmplt_epi16(xmm_ui16_pSrcV[3], xmm_zero));
				r4 = _mm_mullo_epi32(r4, xmm_coeff3);

				r1 = _mm_add_epi32(r1,r2);
				r1 = _mm_add_epi32(r1,r3);
				r1 = _mm_add_epi32(r1,r4);

				//scale down
				r1 = _mm_add_epi32(r1, xmm_iOffset);
				r1  = _mm_srai_epi32( r1, nShift);

				//32bit to 16bit
				r1 = _mm_packs_epi32(r1,r1);

				//clipping
				r1	= _mm_max_epi16(r1, xmm_zero);
				r1	= _mm_min_epi16(r1, xmax);

				_mm_storel_epi64((__m128i *)ui16_pDstV,r1);

				ui16_pSrcU += 4;
				ui16_pSrcV += 4;
				ui16_pDstU += 4;
				ui16_pDstV += 4;


				for( i = (pu_width>>1)-4 ; i > 0; i-- )
				{
					int32 sum_chroma_ver_u = ui16_pSrcU[0]*coeff[0] + ui16_pSrcU[pu_width>>1]*coeff[1] + ui16_pSrcU[2*(pu_width>>1)]*coeff[2] + ui16_pSrcU[3*(pu_width>>1)]*coeff[3];
					int32 sum_chroma_ver_v = ui16_pSrcV[0]*coeff[0] + ui16_pSrcV[pu_width>>1]*coeff[1] + ui16_pSrcV[2*(pu_width>>1)]*coeff[2] + ui16_pSrcV[3*(pu_width>>1)]*coeff[3];

					*ui16_pDstU = Clip3(int32, 0, ((1<<8)-1), (sum_chroma_ver_u + iOffset) >> (nShift));       
					*ui16_pDstV = Clip3(int32, 0, ((1<<8)-1), (sum_chroma_ver_v + iOffset) >> (nShift));

					ui16_pSrcU++;
					ui16_pSrcV++;
					ui16_pDstU++;
					ui16_pDstV++;
				}

			}
		}
		else 
		{
			for( j = y_position_UV*2; j < (y_position_UV*2)+(pu_height>>1); j++ )
			{
				int16* ui8_pDstU0;
				int16* ui8_pDstV0;
				int32 y = Clip3(int32, topStartC, bottomEndC - 1, j);

				refPos16 = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
				phase    = refPos16 & 15;
				refPos   = refPos16 >> 4;
				coeff = i16_chroma_fixed_filter[phase];

#if KW265_TILE
				ui16_pSrcU = ui16_pTempBufU[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
				ui16_pSrcV = ui16_pTempBufV[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#else
				ui16_pSrcU = ui16_pTempBufU  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
				ui16_pSrcV = ui16_pTempBufV  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#endif

				ui8_pDstU0 = ui16_pDstBufU + (j-(y_position_UV*2))*(pu_width>>1);
				ui8_pDstV0 = ui16_pDstBufV + (j-(y_position_UV*2))*(pu_width>>1);

				ui16_pDstU = ui8_pDstU0 + leftOffset;
				ui16_pDstV = ui8_pDstV0 + leftOffset;
				ui16_pSrcU += leftOffset;
				ui16_pSrcV += leftOffset;

		


				for( i = pu_width>>1 ; i > 0; i -= 4 )
				{
					xmm_ui16_pSrcU[0] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 0*(pu_width>>1)));
					xmm_ui16_pSrcU[1] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 1*(pu_width>>1)));
					xmm_ui16_pSrcU[2] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 2*(pu_width>>1)));
					xmm_ui16_pSrcU[3] = _mm_loadu_si128((__m128i *) (ui16_pSrcU + 3*(pu_width>>1)));

					xmm_coeff0 = _mm_set1_epi32(coeff[0]);
					xmm_coeff1 = _mm_set1_epi32(coeff[1]);
					xmm_coeff2 = _mm_set1_epi32(coeff[2]);
					xmm_coeff3 = _mm_set1_epi32(coeff[3]);

					r1 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[0], _mm_cmplt_epi16(xmm_ui16_pSrcU[0], xmm_zero));
					r1 = _mm_mullo_epi32(r1, xmm_coeff0);

					r2 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[1], _mm_cmplt_epi16(xmm_ui16_pSrcU[1], xmm_zero));
					r2 = _mm_mullo_epi32(r2, xmm_coeff1);

					r3 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[2], _mm_cmplt_epi16(xmm_ui16_pSrcU[2], xmm_zero));
					r3 = _mm_mullo_epi32(r3, xmm_coeff2);

					r4 = _mm_unpacklo_epi16(xmm_ui16_pSrcU[3], _mm_cmplt_epi16(xmm_ui16_pSrcU[3], xmm_zero));
					r4 = _mm_mullo_epi32(r4, xmm_coeff3);

					r1 = _mm_add_epi32(r1,r2);
					r1 = _mm_add_epi32(r1,r3);
					r1 = _mm_add_epi32(r1,r4);

					//scale down
					r1 = _mm_add_epi32(r1, xmm_iOffset);
					r1  = _mm_srai_epi32( r1, nShift);

					//32bit to 16bit
					r1 = _mm_packs_epi32(r1,r1);

					//clipping
					r1	= _mm_max_epi16(r1, xmm_zero);
					r1	= _mm_min_epi16(r1, xmax);

					_mm_storel_epi64((__m128i *)ui16_pDstU,r1);

					xmm_ui16_pSrcV[0] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 0*(pu_width>>1)));
					xmm_ui16_pSrcV[1] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 1*(pu_width>>1)));
					xmm_ui16_pSrcV[2] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 2*(pu_width>>1)));
					xmm_ui16_pSrcV[3] = _mm_loadu_si128((__m128i *) (ui16_pSrcV + 3*(pu_width>>1)));


					r1 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[0], _mm_cmplt_epi16(xmm_ui16_pSrcV[0], xmm_zero));
					r1 = _mm_mullo_epi32(r1, xmm_coeff0);

					r2 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[1], _mm_cmplt_epi16(xmm_ui16_pSrcV[1], xmm_zero));
					r2 = _mm_mullo_epi32(r2, xmm_coeff1);

					r3 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[2], _mm_cmplt_epi16(xmm_ui16_pSrcV[2], xmm_zero));
					r3 = _mm_mullo_epi32(r3, xmm_coeff2);

					r4 = _mm_unpacklo_epi16(xmm_ui16_pSrcV[3], _mm_cmplt_epi16(xmm_ui16_pSrcV[3], xmm_zero));
					r4 = _mm_mullo_epi32(r4, xmm_coeff3);

					r1 = _mm_add_epi32(r1,r2);
					r1 = _mm_add_epi32(r1,r3);
					r1 = _mm_add_epi32(r1,r4);

					//scale down
					r1 = _mm_add_epi32(r1, xmm_iOffset);
					r1  = _mm_srai_epi32( r1, nShift);

					//32bit to 16bit
					r1 = _mm_packs_epi32(r1,r1);

					//clipping
					r1	= _mm_max_epi16(r1, xmm_zero);
					r1	= _mm_min_epi16(r1, xmax);

					_mm_storel_epi64((__m128i *)ui16_pDstV,r1);

					ui16_pSrcU += 4;
					ui16_pSrcV += 4;
					ui16_pDstU += 4;
					ui16_pDstV += 4;
				}
			}

		}

		i16pDstY = upsample_Y;
		ui16_pDstU = upsample_Cb;
		ui16_pDstV = upsample_Cr;
	}
#else
	{
		//upsampling
		int32 refPos16 = 0;
		int32 phase    = 0;
		int32 refPos   = 0;
		int32* coeff;

		int32 refPos16_y = 0;
		int32 refPos_y = 0;
		int32 refPos16_uv = 0;
		int32 refPos_uv =0;

		int32   shiftX = 16;
		int32   shiftY = 16;

		int32 shiftXM4 ;
		int32 shiftYM4 ;

		int32 leftStartL  ;
		int32 rightEndL   ;
		int32 topStartL   ;
		int32 bottomEndL  ;
		int32 leftOffset  ;

		int32 shift1      ;

		int32 nShift;
		int32 iOffset;

		int32 leftStartC ;
		int32 rightEndC  ;
		int32 topStartC  ;
		int32 bottomEndC ;

		int32 phaseXC;
		int32 phaseYC;

		//for Luma, if Phase 0, then both PhaseX  and PhaseY should be 0. If symmetric: both PhaseX and PhaseY should be 2
		int32   phaseX = 2 * phase_align_flag;
		int32   phaseY = 2 * phase_align_flag;

		int32   addX = ( ( phaseX * i32_scaleX + 2 ) >> 2 ) + ( 1 << ( shiftX - 5 ) );
		int32   addY = ( ( phaseY * i32_scaleY + 2 ) >> 2 ) + ( 1 << ( shiftY - 5 ) );

		int32   deltaX = (int32)phase_align_flag <<3;
		int32   deltaY = (((int32)phase_align_flag <<3)>>(int32)b_vertPhasePositionEnableFlag) + ((int32)b_vertPhasePositionFlag<<3);

		int32 n=0;
		int32 n1=0;

// 		for ( i = 0; i < 16; i++)
// 		{
// 			memcpy(   i32_luma_filter[i],   i32_luma_fixed_filter[i], sizeof(int32) * NTAPS_US_LUMA   );
// 			memcpy( i32_chroma_filter[i], i32_chroma_fixed_filter[i], sizeof(int32) * NTAPS_US_CHROMA );
// 		}

		deltaX -= ( conformance_bl.left_crop_offset * xScal ) << 4;
		deltaY -= ( conformance_bl.top_crop_offset * yScal  ) << 4;

		shiftXM4 = shiftX - 4;
		shiftYM4 = shiftY - 4;

		i32_strideBL  = cur_slice->pic_size->cwidth + (cur_slice->pic->m_pcFullPelBaseRec[0]->luma_margin_x)*2 ;

		leftStartL = sclaed_el.left_crop_offset;
		rightEndL  = cur_slice->pic_size->width;
		topStartL  = sclaed_el.top_crop_offset;
		bottomEndL = cur_slice->pic_size->height;
		leftOffset = leftStartL > 0 ? leftStartL : 0;

		// g_bitDepthY was set to EL bit-depth, but shift1 should be calculated using BL bit-depth
		shift1 = 0; // g_bitDepthYLayer[refLayerId] - 8;

		if( cur_slice->slice->pps->nCGSFlag )
		{
			shift1 = cur_slice->slice->pps->nCGSOutputBitDepthY - 8;
		}


		//========== horizontal upsampling ===========
		for( i = x_position*2; i < (x_position*2)+pu_width; i++ )
		{
			int32 x = Clip3( int32, leftStartL, rightEndL - 1, i );
			int32 y = Clip3(int32, topStartL, bottomEndL - 1, y_position*2);

			refPos16 = (((x - leftStartL)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16 & 15;
			refPos   = refPos16 >> 4;
			coeff = i32_luma_fixed_filter[phase];

			refPos16_y = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_y   = refPos16_y >> 4;

			i16_pSrcY = ui16_pSrcBufY + refPos -((NTAPS_US_LUMA>>1) - 1);
			i16_pSrcY += ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*(i32_strideBL));

#if KW265_TILE
			i16pDstY = ui16_pTempBufY[tile_idx] +n;   
#else
			i16pDstY = ui16_pTempBufY +n;   
#endif

			for( j = 0; j < (pu_height/2)+7 ; j++ )
			{
				*i16pDstY = (i16_pSrcY[0]*coeff[0] + i16_pSrcY[1]*coeff[1] + i16_pSrcY[2]*coeff[2] + i16_pSrcY[3]*coeff[3] + i16_pSrcY[4]*coeff[4] + i16_pSrcY[5]*coeff[5] + i16_pSrcY[6]*coeff[6] + i16_pSrcY[7]*coeff[7]) >> shift1;
				i16_pSrcY += i32_strideBL;
				i16pDstY += pu_width;
			}
			n++;
		}

		//========== vertical upsampling ===========
		nShift = 12; // 20 - g_bitDepthYLayer[currLayerId];
		iOffset = 1 << (nShift - 1);

		for( j = y_position*2; j < (y_position*2)+pu_height; j++ )
		{
			int16* i16_pDstY0;
			int32 y = Clip3(int32, topStartL, bottomEndL - 1, j);

			refPos16 = ((( y - topStartL )*i32_scaleY + addY) >> shiftYM4) - deltaY;
			phase    = refPos16 & 15;
			refPos   = refPos16 >> 4;
			coeff = i32_luma_fixed_filter[phase];

#if KW265_TILE
			i16_pSrcY = ui16_pTempBufY[tile_idx] + (refPos -((NTAPS_US_LUMA>>1) - 1))*pu_width - ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*pu_width);  //refPos -((NTAPS_US_LUMA>>1) - 1))*i32_strideEL
#else
			i16_pSrcY = ui16_pTempBufY + (refPos -((NTAPS_US_LUMA>>1) - 1))*pu_width - ((refPos_y -((NTAPS_US_LUMA>>1) - 1))*pu_width);  //refPos -((NTAPS_US_LUMA>>1) - 1))*i32_strideEL
#endif
			i16_pDstY0 = ui16_pDstBufY + (j-(y_position*2)) * pu_width;

			i16pDstY = i16_pDstY0 + leftOffset;
			i16_pSrcY += leftOffset;

			for( i = pu_width; i > 0; i-- )
			{

				int32 sum_luma_ver = i16_pSrcY[0]*coeff[0] + i16_pSrcY[pu_width]*coeff[1] + i16_pSrcY[2*pu_width]*coeff[2] + i16_pSrcY[3*pu_width]*coeff[3] + i16_pSrcY[4*pu_width]*coeff[4] + i16_pSrcY[5*pu_width]*coeff[5] + i16_pSrcY[6*pu_width]*coeff[6] + i16_pSrcY[7*pu_width]*coeff[7];        
				*i16pDstY = Clip3(int32, 0, ((1<<8)-1), (sum_luma_ver + iOffset) >> nShift );
				i16_pSrcY++;
				i16pDstY++;
			}
		}


		//========== UV component upsampling ===========

		i32_strideBL  = (cur_slice->pic_size->cwidth)/2 + (cur_slice->pic->m_pcFullPelBaseRec[0]->chroma_margin_x)*2;

		leftStartC = sclaed_el.left_crop_offset >> 1;
		rightEndC  = cur_slice->pic_size->width >> 1;
		topStartC  = sclaed_el.top_crop_offset;
		bottomEndC = cur_slice->pic_size->height >> 1;
		leftOffset = leftStartC > 0 ? leftStartC : 0;
		shiftX = 16;
		shiftY = 16;

		phaseXC = phase_align_flag;
		phaseYC = phase_align_flag + 1;

		addX       = ( ( phaseXC * i32_scaleX + 2 ) >> 2 ) + ( 1 << ( shiftX - 5 ) );
		addY       = ( ( phaseYC * i32_scaleY + 2 ) >> 2 ) + ( 1 << ( shiftY - 5 ) );

		deltaX     = (int32)phase_align_flag << 2;
		deltaY     = ((( (int32)phase_align_flag +1)<<2)>>(int32)b_vertPhasePositionEnableFlag)+((int32)b_vertPhasePositionFlag<<3);

		deltaX  -= ( ( conformance_bl.left_crop_offset * xScal ) >> 1 ) << 4;
		deltaY  -= ( ( conformance_bl.top_crop_offset  * yScal ) >> 1 ) << 4;

		shiftXM4 = shiftX - 4;
		shiftYM4 = shiftY - 4;

		// g_bitDepthC was set to EL bit-depth, but shift1 should be calculated using BL bit-depth
		shift1 = 0; // g_bitDepthCLayer[refLayerId] - 8;

		if( cur_slice->slice->pps->nCGSFlag )
		{
			shift1 = cur_slice->slice->pps->nCGSOutputBitDepthC- 8;
		}

		//========== horizontal upsampling ===========
		for( i = x_position_UV*2; i < (x_position_UV*2)+(pu_width>>1); i++ )
		{
			int32 x = Clip3(int32, leftStartC, rightEndC - 1, i);
			int32 y = Clip3(int32, topStartC, bottomEndC - 1, y_position_UV*2);

			refPos16 = (((x - leftStartC)*i32_scaleX + addX) >> shiftXM4) - deltaX;
			phase    = refPos16 & 15;
			refPos   = refPos16 >> 4;
			coeff = i32_chroma_fixed_filter[phase];

			refPos16_uv = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
			refPos_uv   = refPos16_uv >> 4;

			ui16_pSrcU = ui16_pSrcBufU + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcU += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));     //1040*y_position_UV +
			ui16_pSrcV = ui16_pSrcBufV + refPos -((NTAPS_US_CHROMA>>1) - 1);
			ui16_pSrcV += ((refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(i32_strideBL));
#if KW265_TILE
			ui16_pDstU = ui16_pTempBufU[tile_idx] +n1;
			ui16_pDstV = ui16_pTempBufV[tile_idx] +n1;
#else
			ui16_pDstU = ui16_pTempBufU +n1;
			ui16_pDstV = ui16_pTempBufV +n1;
#endif

			for( j = 0; j < ((pu_height>>1)/2)+4 ; j++ )
			{
				int32 sum_chroma_hor_u = ui16_pSrcU[0]*coeff[0] + ui16_pSrcU[1]*coeff[1] + ui16_pSrcU[2]*coeff[2] + ui16_pSrcU[3]*coeff[3];
				int32 sum_chroma_hor_v = ui16_pSrcV[0]*coeff[0] + ui16_pSrcV[1]*coeff[1] + ui16_pSrcV[2]*coeff[2] + ui16_pSrcV[3]*coeff[3];

				*ui16_pDstU = sum_chroma_hor_u >> shift1;
				*ui16_pDstV = sum_chroma_hor_v >> shift1;

				ui16_pSrcU += i32_strideBL;
				ui16_pSrcV += i32_strideBL;
				ui16_pDstU += pu_width>>1;
				ui16_pDstV += pu_width>>1;
			}
			n1++;
		}


		//========== vertical upsampling ===========

		nShift = 12; // 20 - g_bitDepthCLayer[currLayerId];

		iOffset = 1 << (nShift - 1);

		for( j = y_position_UV*2; j < (y_position_UV*2)+(pu_height>>1); j++ )
		{
			int16* ui8_pDstU0;
			int16* ui8_pDstV0;
			int32 y = Clip3(int32, topStartC, bottomEndC - 1, j);
			refPos16 = (((y - topStartC)*i32_scaleY + addY) >> shiftYM4) - deltaY;
			phase    = refPos16 & 15;
			refPos   = refPos16 >> 4;
			coeff = i32_chroma_fixed_filter[phase];

#if KW265_TILE
			ui16_pSrcU = ui16_pTempBufU[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
			ui16_pSrcV = ui16_pTempBufV[tile_idx]  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#else
			ui16_pSrcU = ui16_pTempBufU  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
			ui16_pSrcV = ui16_pTempBufV  + (refPos -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1) - (refPos_uv -((NTAPS_US_CHROMA>>1) - 1))*(pu_width>>1);
#endif

			ui8_pDstU0 = ui16_pDstBufU + (j-(y_position_UV*2))*(pu_width>>1);
			ui8_pDstV0 = ui16_pDstBufV + (j-(y_position_UV*2))*(pu_width>>1);

			ui16_pDstU = ui8_pDstU0 + leftOffset;
			ui16_pDstV = ui8_pDstV0 + leftOffset;
			ui16_pSrcU += leftOffset;
			ui16_pSrcV += leftOffset;

			for( i = pu_width>>1 ; i > 0; i-- )
			{
				int32 sum_chroma_ver_u = ui16_pSrcU[0]*coeff[0] + ui16_pSrcU[pu_width>>1]*coeff[1] + ui16_pSrcU[2*(pu_width>>1)]*coeff[2] + ui16_pSrcU[3*(pu_width>>1)]*coeff[3];
				int32 sum_chroma_ver_v = ui16_pSrcV[0]*coeff[0] + ui16_pSrcV[pu_width>>1]*coeff[1] + ui16_pSrcV[2*(pu_width>>1)]*coeff[2] + ui16_pSrcV[3*(pu_width>>1)]*coeff[3];

				*ui16_pDstU = Clip3(int32, 0, ((1<<8)-1), (sum_chroma_ver_u + iOffset) >> (nShift));       //여기서 뻑이가요
				*ui16_pDstV = Clip3(int32, 0, ((1<<8)-1), (sum_chroma_ver_v + iOffset) >> (nShift));

				ui16_pSrcU++;
				ui16_pSrcV++;
				ui16_pDstU++;
				ui16_pDstV++;
			}

		}


		i16pDstY = upsample_Y;
		ui16_pDstU = upsample_Cb;
		ui16_pDstV = upsample_Cr;

	}
#endif
}

#endif
