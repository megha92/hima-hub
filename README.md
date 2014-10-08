hima-hub

 * @file
 * Kate subtitles format demuxer
 */

#include "avformat.h"
#include "internal.h"
#include "subtitles.h"

typedef struct {
    FFDemuxSubtitlesQueue q;
} KateContext;

static int kate_probe(AVProbeData *p)
{
    char c;
    const unsigned char *ptr = p->buf;

    if (  sscanf(ptr," %d:%d:%d --> %d:%d:%d",&v ))
        return AVPROBE_SCORE_MAX;
    return 0;
}

static int64_t get_pts(const char **buf, int *duration,
                       int32_t *x1, int32_t *y1, int32_t *x2, int32_t *y2)
{
   
        int hh1, mm1, ss1, ms1;
        int hh2, mm2, ss2, ms2;
        if(sscanf(*buf,  " %d:%d:%d --> %d:%d:%d",&hh1, &mm1, &ss1,&hh2, &mm2, &ss2)>=4) 
       {
            int64_t start = (hh1*3600LL + mm1*60LL + ss1) * 1000LL;
            int64_t end   = (hh2*3600LL + mm2*60LL + ss2) * 1000LL;
            *duration = end - start;                                                                        
            *buf += ff_subtitles_next_line(*buf);
           
      }
    return AV_NOPTS_VALUE;
}


static int kate_read_header(AVFormatContext *s)
{
    KateContext *kate = s->priv_data;
    AVStream *st = avformat_new_stream(s, NULL);

    if (!st)
        return AVERROR(ENOMEM);
    avpriv_set_pts_info(st, 64, 1, 100);
    st->codec->codec_type = AVMEDIA_TYPE_SUBTITLE;
    st->codec->codec_id   = AV_CODEC_ID_TEXT;

    while (!avio_feof(s->pb)) {
        char line[4096];                                                                                  
        char *p = line;
        const int64_t pos = avio_tell(s->pb);
        int len = ff_get_line(s->pb, line, sizeof(line));
        int64_t pts_start;
        int duration;
        int32_t x1 = -1, y1 = -1, x2 = -1, y2 = -1;
        if (!len)
            break;

        line[strcspn(line, "\r\n")] = 0;

        pts_start = get_pts(s->pb,duration,x1,y1,x2,y2);
        if (pts_start != AV_NOPTS_VALUE) {
            AVPacket *sub;

            sub = ff_subtitles_queue_insert(&kate->q, p, strlen(p), 0);
            if (!sub)
                return AVERROR(ENOMEM);
            sub->pos = pos;
            sub->pts = pts_start;
            sub->duration = -1;
        }
    }

    ff_subtitles_queue_finalize(&kate->q);
    return 0;
}

static int kate_read_packet(AVFormatContext *s, AVPacket *pkt)
{
    KateContext *kate = s->priv_data;
    return ff_subtitles_queue_read_packet(&kate->q, pkt);
}

static int kate_read_seek(AVFormatContext *s, int stream_index,
                             int64_t min_ts, int64_t ts, int64_t max_ts, int flags)
{
    KateContext *kate = s->priv_data;
    return ff_subtitles_queue_seek(&kate->q, s, stream_index,
                                   min_ts, ts, max_ts, flags);
}

static int kate_read_close(AVFormatContext *s)
{
    KateContext *kate = s->priv_data;
    ff_subtitles_queue_clean(&kate->q);
    return 0;
}

AVInputFormat ff_kate_demuxer = {
    .name           = "kate",
    .long_name      = NULL_IF_CONFIG_SMALL("Kate subtitles"),
    .priv_data_size = sizeof(KateContext),
    .read_probe     = kate_probe,
    .read_header    = kate_read_header,
    .read_packet    = kate_read_packet,
    .read_seek2     = kate_read_seek,
    .read_close     = kate_read_close,
    .extensions     = "txt",
};===
