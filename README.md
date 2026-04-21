# msu2draftnotes
notes towards maybe a draft of a msu-2 spec, maybe

```
i/o mode 0 read (unchanged)
$2000	trackA_status
$2001	data_read
$2002	"S"
$2003	"-"
$2004	"M"
$2005	"S"
$2006	"U"
$2007	"1"

trackA_status bit 0		revision bit 0
trackA_status bit 1		revision bit 1
trackA_status bit 2		revision bit 2
trackA_status bit 3		trackA_missing
trackA_status bit 4		trackA_playing
trackA_status bit 5		trackA_repeat
trackA_status bit 6		trackA_busy
trackA_status bit 7		data_busy

i/o mode 1 read (NEW)
$2000	trackA_status
$2001	unified_audio_status_pt1
$2002	unified_audio_status_pt2
$2003	unified_audio_status_pt3
$2004	info_out << 0
$2005	info_out << 8
$2006	info_out << 16
$2007	info_out << 24

unified_audio_status_pt1 bit 0		trackA_missing
unified_audio_status_pt1 bit 1		trackB_missing
unified_audio_status_pt1 bit 2		trackC_missing
unified_audio_status_pt1 bit 3		trackD_missing
unified_audio_status_pt1 bit 4		trackA_playing
unified_audio_status_pt1 bit 5		trackB_playing
unified_audio_status_pt1 bit 6		trackC_playing
unified_audio_status_pt1 bit 7		trackD_playing

unified_audio_status_pt2 bit 0		trackA_repeat
unified_audio_status_pt2 bit 1		trackB_repeat
unified_audio_status_pt2 bit 2		trackC_repeat
unified_audio_status_pt2 bit 3		trackD_repeat
unified_audio_status_pt2 bit 4		trackA_busy
unified_audio_status_pt2 bit 5		trackB_busy
unified_audio_status_pt2 bit 6		trackC_busy
unified_audio_status_pt2 bit 7		trackD_busy

unified_audio_status_pt3 bit 0		trackA_fadein
unified_audio_status_pt3 bit 1		trackB_fadein
unified_audio_status_pt3 bit 2		trackC_fadein
unified_audio_status_pt3 bit 3		trackD_fadein
unified_audio_status_pt3 bit 4		trackA_fadeout
unified_audio_status_pt3 bit 5		trackB_fadeout
unified_audio_status_pt3 bit 6		trackC_fadeout
unified_audio_status_pt3 bit 7		trackD_fadeout
```

`trackX_fadein` and `trackX_fadeout` return 1 if the track is currently in the middle of a fade in or fade out. Else 0.

`info_out` returns a value requested via `extra_control.info_type`  
See below

```
i/o mode 0 write (unchanged except for previous unused bits in control)
$2000	data_seek << 0
$2001	data_seek << 8
$2002	data_seek << 16
$2003	data_seek << 24
$2004	audio_track << 0
$2005	audio_track << 8
$2006	audio_volume
$2007	control

control bit 0	play
control bit 1	repeat
control bit 2	bsnes_resume
control bit 3	auto_promote (NEW)
control bit 4	swap_tracks (NEW)
control bit 5	io_mode_param << 0 (NEW)
control bit 6	io_mode_param << 1 (NEW)
control bit 7	io_mode_select (NEW)
```

If `io_mode_select == 1`, then set i/o mode according to `io_mode_param` (and IGNORE all other controls bits).  
This draft only uses io modes 0 and 1.  
I/O modes 2 and 3 are reserved for future use.

If `swap_tracks == 1`, then swap the handles of the current trackA and the track selected by `extra_control.which_track` (and IGNORE all other control bits).  
(The intent of this feature is to allow the user to set up a cross-fade, then swap the handles, always leaving the ongoing active music track -- which is the one future commands are most likely to care about -- as trackA.)

`auto_promote` sets a flag on currently selected track to automatically swap its handle to the next-higher handle when the track with the next-higher handle terminates by reaching its end or by finishing a fade out, or swaps to a higher handle itself. (Example: If tracks A, B, and C are playing, and tracks B and C have the `auto_promote` flag, then when track A finishes, track B will change its handle to track A and track C will change its handle to track B.) When promoting, tracks without the `auto_promote` flag may be jumped over. (Example: If tracks A, B, and C are playing, and only track C has the auto_promote flag, then when track A finishes, track C will change its handle to track A.) (The intent of this feature is a slightly more advanced way to shuffle the ongoing active music track towards the trackA handle, and also shuffle free space towards the trackD handle. Though I worry that maybe this is too complicated and no one would use it.)

```
i/o mode 1 write (NEW)
$2000	extra_control
$2001	fade_step << 8
$2002	fade_step << 16
$2003	fade_to_volume << 24
$2004	audio_track << 0
$2005	audio_track << 8
$2006	audio_volume
$2007	control
```

(If we needed even more space, we could change the behavior of control bits 0-4 in higher i/o modes.)

`fade-step` is volume change per 32kHz sample.  
Not sure this is ideal.  
Could alternatively use the number of 32kHz samples over which to apply the fade.  
Or volume change per millisecond.  
Or milliseconds over which to apply the fade.  
Or even the number of master clock cycles over which to apply the fade.  

`fade_to_volume` is the target volume at the end of a fade.

```
extra_control bit 0		which_track << 0
extra_control bit 1		which_track << 1
extra_control bit 2		fade
extra_control bit 3		fade_direction
extra_control bit 4		fade_cancel
extra_control bit 5		info_type << 0
extra_control bit 6		info_type << 1
extra_control bit 7		info_type << 2
```

`which_track` controls whether the following are applied to trackA, B, C, or D:
- audio_track
- audio_volume
- control.play
- control.repeat
- control.bsnes_resume
- control.auto_swap
- extra_control.fade
- extra_control.fade_direction
- extra_control.fade_cancel
- fade_step
- fade_to_volume
- info_out (whether info_out displays for trackA, B, C, or D)

When i/o mode is 0, `which_track` is assumed to be 0 (track A).

`fade` sets the fade flag for the track selected by which_track.

`fade_cancel` clears the fade flag from the track selected by which_track, leaving it at whatever its current volume is. (Not sure if anyne would use this.)

`fade_direction` sets whether the fade applied to the track selected by which_track is a fade-in (1) or a fade-out (0).

info_type sets what kind of information to output to info_out (combined with which_track setting which track's information to use).
- 0 - current position in 32kHz samples
- 1 - track length in 32kHz samples
- 2 - track completion in 1/4294967295ths
- 3 - 2 values:
     - track completion in 1/65535ths in $2004 and $2005
     - fade completion in 1/65535ths in $2006 and $2007
- 4 - fade remaining in 32kHz samples
- 5 - unused
- 6 - unused
- 7 - unused
- 8 - unused

Not completely sure 32kHz samples are the best unit here.  
Might want to use milliseconds or master clock ticks instead?  
(44.1kHz samples doesn't seem very useful since nothing in the original rom code will readily convert to it. Better to do that conversion on the MSU-1 side.)
