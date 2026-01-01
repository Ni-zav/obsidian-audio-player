<template>
  <div class="audio-player-ui" tabindex="-1" 
       @click="handlePlayerClick"
       @keydown="handleKeydown"
       @mouseenter="handlePlayerMouseEnter"
       @mouseleave="handlePlayerMouseLeave">
    <div class="player-title">{{ displayTitle }}</div>
    
    <!-- Video wrapper - wraps around controls when video is showing -->
    <div class="player-main" :class="{ 'video-mode': isVideo && showVideoEmbed }" 
         @mouseenter="isVideoHovered = true" @mouseleave="isVideoHovered = false">
      <!-- Video element (behind controls when showing) -->
      <video 
        v-if="isVideo"
        v-show="showVideoEmbed"
        ref="videoElement"
        class="video-player"
        :src="srcPath"
        @timeupdate="videoTimeUpdateHandler"
        @loadedmetadata="videoLoadedHandler"
        @play="videoPlayHandler"
        @pause="videoPauseHandler"
        @click="togglePlay"
      ></video>
      
      <!-- Controls overlay (always visible, on top of video when showing) -->
      <div class="player-controls-wrapper" :class="{ 
        'on-video': isVideo && showVideoEmbed,
        'video-controls-dimmed': isDimmed
      }">
        <!-- Top row: Play controls + Eye toggle -->
        <div class="controls-row">
          <div class="controls-left">
            <div class="control-btn control-small" @click="skipToBeginning" ref="skipBackButton"></div>
            <div class="control-btn control-play" @click="togglePlay" ref="playpause"></div>
            <div class="control-btn control-small" @click="toggleLooping" :class="{ 'looping': looping }" ref="loopButton"></div>
            <div class="inline-timeline">
              <span class="current-time" @mouseover="setCopyTimestampTooltip" @click="copyTimestampToClipboard" ref="currentTime">
                {{ displayedCurrentTime }}
              </span>
              <span class="time-separator">/</span>
              <span class="duration">
                {{ displayedDuration }}
              </span>
            </div>
          </div>
          <div class="controls-right" v-if="isVideo">
            <div class="control-btn control-small control-eye" @click="toggleVideoEmbed" ref="videoToggleButton">
              <span ref="videoToggleIcon"></span>
            </div>
          </div>
        </div>
        
        <!-- Waveform container with two layers -->
        <div class="waveform-container">
          <!-- Background layer: Comment shading (behind audio bars) -->
          <div class="waveform waveform-comments">
            <div class="wv-comment-wrapper" v-for="(s, i) in filteredData" :key="'cmt-' + i">
              <!-- Multiple shade layers per bar - one for each comment, using rank for consistent heights -->
              <div v-if="barsWithComments.includes(i)"
                v-for="(cmt, ci) in commentsForBar(i)" 
                :key="ci"
                class="wv-comment-shade"
                :style="{ 
                  height: (100 - cmt.rank * 5) + '%',
                  opacity: 0.25 + cmt.rank * 0.08
                }">
              </div>
            </div>
          </div>
          
          <!-- Foreground layer: Audio waveform (variable height, on top) -->
          <div class="waveform waveform-audio" ref="waveformAudio" 
               @mousemove="handleWaveformMouseMove($event)" 
               @mouseout="unhighlightComment(); hideWaveformTooltip();"
               @mousedown="handleWaveformMouseDown($event)">
            <div class="wv" v-for="(s, i) in filteredData" :key="srcPath + i" 
                 :class="{ 'highlighted': highlightedBars.includes(i) }" 
                 :style="getBarStyle(s, i)">
            </div>
          </div>
        </div>
        <div class="moodbar" v-html="displayedMoodbar"></div>
        

      </div>
    </div>
    
    <!-- Comment list -->
    <div class="comment-list">
      <AudioCommentVue ref="audiocomment" v-for="cmt in commentsSorted" :class="{
        'active-comment': cmt == activeComment,
        'active-parent-comment': isActiveParentComment(cmt),
        'current-comment': cmt == currentComment,
        'highlighted-comment': cmt == highlightedComment
      }" @move-playhead="setPlayheadSecs" @mouseover="highlightBars(barsForComment(cmt))"
        @mouseout="unhighlightBars()" :cmt="cmt" :key="cmt.timeString"></AudioCommentVue>
      <div class="comment" v-if="commentsSorted.length > 0" :class="{
        'current-comment': currentComment == null
      }">
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { TFile, setIcon, setTooltip, MarkdownPostProcessorContext } from 'obsidian'
import { defineComponent, PropType } from 'vue';
import { AudioComment } from '../types'
import { secondsToString, secondsToNumber, range, hasOverlap } from '../utils'

import AudioCommentVue from './AudioComment.vue';

export default defineComponent({
  name: 'App',
  components: {
    AudioCommentVue
  },
  props: {
    filepath: String,
    ctx: Object as PropType<MarkdownPostProcessorContext>,
    title: String,
    content: Object as PropType<HTMLElement>,
    moodbar: Object as PropType<HTMLElement>,
    mdElement: Object as PropType<HTMLElement>,
    audio: Object as PropType<HTMLAudioElement>,
    isVideo: Boolean
  },
  data() {
    return {
      items: [...Array(100).keys()],
      srcPath: '',

      filteredData: [] as number[],
      rawAudioData: null as Float32Array | null,  // Store raw data for reprocessing
      nSamples: 100,
      duration: 0,
      currentTime: 0,
      playing: false,
      looping: false,
      button: undefined as HTMLSpanElement | undefined,

      newComment: '',
      comments: [] as AudioComment[],
      activeComment: null as AudioComment | null,
      currentComment: null as AudioComment | null,
      highlightedComment: null as AudioComment | null,
      highlightedBars: [] as number[],

      ro: ResizeObserver,
      smallSize: false,
      
      // Video-specific state
      showVideoEmbed: false,
      isVideoHovered: false,
      
      // Waveform tooltip state for realistic hover time
      waveformTooltipEl: null as HTMLElement | null,
      waveformTooltipTimeout: null as ReturnType<typeof setTimeout> | null,
      
      // Smooth time update animation frame
      animationFrameId: null as number | null,
      
      // Waveform scrubbing state
      isScrubbing: false,
      wasPlayingBeforeScrub: false,
      
      // Player hover state for keyboard shortcuts
      isPlayerHovered: false,
    }
  },
  computed: {
    displayTitle() { return this.title; },
    displayedCurrentTime() { return secondsToString(Math.min(this.currentTime, this.duration)); },
    displayedDuration() { return secondsToString(this.duration); },
    displayedMoodbar() { return this.moodbar?.outerHTML; },
    currentBar() { return this.barForTime(this.currentTime); },
    startBars() { return this.commentsSorted.map((c: AudioComment) => c.barEdges[0]) },
    endBars() { return this.commentsSorted.map((c: AudioComment) => c.barEdges[1]) },
    barsWithComments() { return this.comments.map((c: AudioComment) => range(...c.barEdges)).flat().unique(); },
    commentsSorted() { return this.comments.sort((x: AudioComment, y: AudioComment) => x.timeStart - y.timeStart); },
    
    // Check if controls should be dimmed (video mode + playing + not hovered)
    isDimmed() {
      return this.isVideo && this.showVideoEmbed && this.playing && !this.isVideoHovered;
    },
    
    // Calculate optimal bar count based on audio duration only (consistent, no resize issues)
    optimalBarCount() {
      if (this.duration <= 0) return 100; // default
      
      if (this.duration < 30) {
        // Short audio (<30s): ~100 bars (1 bar per 0.3 sec)
        return Math.max(50, Math.ceil(this.duration / 0.3));
      } else if (this.duration < 180) {
        // Medium audio (30s-3min): ~150-200 bars
        return Math.min(200, Math.ceil(this.duration / 0.6));
      } else if (this.duration < 600) {
        // Long audio (3-10min): 200-300 bars
        return Math.min(300, Math.ceil(this.duration / 1.5));
      } else {
        // Very long audio (>10min): cap at 400 bars
        return Math.min(400, Math.ceil(this.duration / 3));
      }
    },
    
    // Calculate progress within the current bar (0 to 100%)
    currentBarProgress() {
      if (this.duration <= 0 || this.nSamples <= 0) return 0;
      
      const barDuration = this.duration / this.nSamples;
      const barStartTime = this.currentBar * barDuration;
      const progressInBar = (this.currentTime - barStartTime) / barDuration;
      
      // Clamp between 0 and 100
      return Math.max(0, Math.min(100, progressInBar * 100));
    }
  },
  methods: {
    getSectionInfo() { return this.ctx.getSectionInfo(this.mdElement); },
    getParentWidth() { return this.mdElement.clientWidth },
    isCurrent() { return this.audio.src === this.srcPath; },
    
    // Get style for a waveform bar - uses CSS variable for smooth progress animation
    getBarStyle(s: number, i: number) {
      const height = s * 35 + 'px';
      
      if (i === this.currentBar) {
        // Currently playing bar: set progress variable (0-100)
        return {
          height,
          '--bar-progress': `${this.currentBarProgress}%`
        };
      } else if (i < this.currentBar) {
        // Played bars: set progress to 100% for seamless transition
        return {
          height,
          '--bar-progress': '100%'
        };
      }
      
      // Future bars: progress at 0%
      return { 
        height,
        '--bar-progress': '0%'
      };
    },
    onResize() {
      this.smallSize = this.$el.clientWidth < 300;
      // No longer recalculating bars on resize to avoid uneven gaps
    },
    async loadFile() {
      // read file from vault 
      const file = window.app.vault.getAbstractFileByPath(this.filepath) as TFile;

      // process audio file & set audio el source
      if (file && file instanceof TFile) {
        //check cached values
        if (!this.loadCache())
          this.processAudio(file.path);

        this.srcPath = window.app.vault.getResourcePath(file);
      }
    },
    saveCache() {
      // Cache with current bar count for reference
      localStorage[`${this.filepath}`] = JSON.stringify({
        data: this.filteredData,
        nSamples: this.nSamples
      });
      localStorage[`${this.filepath}_duration`] = this.duration;
    },
    loadCache(): boolean {
      let cachedData = localStorage[`${this.filepath}`];
      let cachedDuration = localStorage[`${this.filepath}_duration`];

      if (!cachedData) { return false; }

      try {
        const cached = JSON.parse(cachedData);
        // Check if cached data matches current bar count (allow some tolerance)
        if (cached.data && cached.nSamples) {
          this.filteredData = cached.data;
          this.nSamples = cached.nSamples;
        } else {
          // Old cache format - still usable
          this.filteredData = cached;
        }
        this.duration = Number.parseFloat(cachedDuration);
        return true;
      } catch {
        return false;
      }
    },
    async processAudio(path: string) {
      const arrBuf = await window.app.vault.adapter.readBinary(path);
      const audioContext = new AudioContext();

      audioContext.decodeAudioData(arrBuf, (buf) => {
        this.rawAudioData = buf.getChannelData(0);
        this.duration = buf.duration;
        
        // Calculate optimal bar count and generate waveform
        this.recalculateBars();
      })
    },
    recalculateBars() {
      if (!this.rawAudioData || !this.duration) return;
      
      // Update nSamples to optimal count
      this.nSamples = this.optimalBarCount;
      
      const tempArray = [] as number[];
      const rawData = this.rawAudioData;
      const blockSize = Math.floor(rawData.length / this.nSamples);
      
      for (let i = 0; i < this.nSamples; i++) {
        let blockStart = blockSize * i;
        let sum = 0;
        for (let j = 0; j < blockSize; j++) {
          sum += Math.abs(rawData[blockStart + j]);
        }
        tempArray.push(sum / blockSize);
      }

      let maxval = Math.max(...tempArray);
      this.filteredData = tempArray.map(x => x / maxval);
      this.saveCache();
    },
    barMouseDownHandler(i: number) {
      let time = i / this.nSamples * this.duration;
      this.setPlayheadSecs(time);
    },
    setPlayheadSecs(time: any) {
      this.currentTime = time;
      if (!this.isCurrent()) {
        this.audio.src = this.srcPath;
      }

      if (this.isCurrent()) {
        this.audio.currentTime = time;
      }
      
      // Sync video if showing
      if (this.showVideoEmbed && this.$refs.videoElement) {
        (this.$refs.videoElement as HTMLVideoElement).currentTime = time;
      }
    },
    toggleLooping() {
      this.looping = !this.looping;
      if (!this.isCurrent()) {
        this.audio.src = this.srcPath;
      }
      this.audio.loop = this.looping;
      
      // Sync video loop state
      if (this.showVideoEmbed && this.$refs.videoElement) {
        (this.$refs.videoElement as HTMLVideoElement).loop = this.looping;
      }
    },
    togglePlay() {
      if (!this.isCurrent()) {
        this.audio.src = this.srcPath;
      }

      if (this.audio.paused) {
        this.globalPause();
        this.play();
      } else {
        this.pause();
      }
    },
    play() {
      if (this.currentTime > 0) {
        this.audio.currentTime = this.currentTime;
      }
      this.audio.addEventListener('timeupdate', this.timeUpdateHandler);
      this.audio?.play();
      this.playing = true;
      this.setBtnIcon('pause');
      
      // Start smooth time updates
      this.startSmoothTimeUpdate();
      
      // Sync video play
      if (this.showVideoEmbed && this.$refs.videoElement) {
        const video = this.$refs.videoElement as HTMLVideoElement;
        video.currentTime = this.audio.currentTime;
        video.play();
      }
    },
    pause() {
      this.audio?.pause();
      this.playing = false;
      this.setBtnIcon('play');
      
      // Stop smooth time updates
      this.stopSmoothTimeUpdate();
      
      // Sync video pause
      if (this.showVideoEmbed && this.$refs.videoElement) {
        (this.$refs.videoElement as HTMLVideoElement).pause();
      }
    },
    globalPause() {
      const ev = new Event('allpause');
      document.dispatchEvent(ev);
    },
    skipToBeginning() {
      if (!this.isCurrent()) {
        this.audio.src = this.srcPath;
      }
      this.audio.currentTime = 0;
      this.currentTime = 0;
      
      // Sync video reset when video embed is showing
      if (this.showVideoEmbed && this.$refs.videoElement) {
        (this.$refs.videoElement as HTMLVideoElement).currentTime = 0;
      }
    },
    timeUpdateHandler() {
      if (this.isCurrent()) {
        this.currentTime = this.audio?.currentTime;

        // Calculate current comment (comment where the current time marker should be displayed)
        // The current comment is the next comment whose END time is after the current time
        if (this.comments.length > 0) {
          const firstComment = this.commentsSorted[0];
          if (this.currentTime <= firstComment.timeStart) {
            this.currentComment = firstComment;
          } else {
            // Find comments whose end time is after current time (not yet finished)
            const upcomingComments = this.commentsSorted.filter((x: AudioComment) =>
              this.currentTime < x.timeEnd
            );
            if (upcomingComments.length > 0) {
              // The current comment is the first one that hasn't ended yet
              this.currentComment = upcomingComments[0];
            } else {
              // All comments have ended
              this.currentComment = null;
            }
          }
        }

        // Calculate active comment (currently playing segment)
        const currentComments = this.commentsSorted.filter((x: AudioComment) =>
          this.currentTime >= x.timeStart && this.currentTime <= x.timeEnd
        );
        if (currentComments.length > 0) {
          const activeComment = currentComments[currentComments.length - 1];
          if (activeComment != this.activeComment) {
            const commentEl = this.$refs.audiocomment[activeComment.index].$el;
            commentEl.scrollIntoView({ block: 'nearest', inline: 'start', behavior: 'smooth' });
          }
          this.activeComment = activeComment;
        } else {
          this.activeComment = null;
        }

        // Check if immediately previous comment should be looped
        const immediatelyPreviousComments = this.commentsSorted.filter((x: AudioComment) =>
          this.currentTime - x.timeEnd >= 0 && this.currentTime - x.timeEnd <= 0.5
        );
        if (immediatelyPreviousComments.length > 0) {
          if (immediatelyPreviousComments[0].looping) {
            this.setPlayheadSecs(immediatelyPreviousComments[0].timeStart);
          }
        }
      }
    },

    setBtnIcon(icon: string) { 
      setIcon(this.button, icon);
    },

    getComments(): Array<AudioComment> {
      const cmtElems = Array.from(this.content?.children || []);

      // parse comments into timestamp/window and comment text
      const timeStampSeparator = ' --- '
      const cmts = cmtElems.map((x: HTMLElement, i) => {
        const cmtParts = x.innerText.split(timeStampSeparator);
        if (cmtParts.length == 2) {
          const timeString = cmtParts[0];
          const timeWindow = timeString.split('-');
          const timeStartStr = timeWindow[0];
          const timeStart = secondsToNumber(timeStartStr);
          let timeEndStr = timeStartStr;
          // by default, timestamps with only start time are assumed to last 1s
          let timeEnd = secondsToNumber(timeEndStr) + 1;
          if (timeWindow.length == 2) {
            timeEndStr = timeWindow[1];
            timeEnd = secondsToNumber(timeEndStr);
          }
          if (!isNaN(timeStart) && !isNaN(timeEnd)) {
            const content = x.innerHTML.replace(timeString + timeStampSeparator, '');
            const bars = [timeStart, timeEnd].map(t => this.barForTime(t)) as [number, number];
            const cmt: AudioComment = {
              timeStart: timeStart,
              timeEnd: timeEnd,
              timeString: timeString,
              content: content,
              index: 0, // calculated at the end when sorting
              barEdges: bars,
              rank: 0, // calculated at the end
              looping: false  // do not loop by default
            }
            return cmt;
          }
        }
      });

      // Calculate comment index and rank
      const allCmts = cmts.filter(Boolean) as Array<AudioComment>;
      // Calculate overlaps between comment time windows.
      // Each comment gets a 'rank', which is used to calculate the
      // height of the comment's highlight on each waveform bin.
      // Rank 0 comments do not have any temporally overlapping
      // previous comments, so their waveform highlight bars will be
      // positioned at the topmost height.
      allCmts.sort((x: AudioComment, y: AudioComment) =>
        x.timeStart - y.timeStart
      ).forEach((cmt: AudioComment, i: number) => {
        // First comment has rank 0, because it cannot overlap with
        // any preceding comment by definition
        if (i == 0) return;
        cmt.index = i;
        const overlaps = allCmts.slice(0, i)
          .filter(c => hasOverlap(c.barEdges, cmt.barEdges));
        if (overlaps.length > 0) {
          while (overlaps.filter(c => c.rank == cmt.rank).length != 0) {
            // Increase rank after each successive overlap with a
            // previous comment
            cmt.rank += 1;
          }
        }
      });
      return allCmts;
    },
    barForTime(t: number) { return Math.floor(t / this.duration * this.nSamples); },
    barsForComment(cmt: AudioComment) { return range(...cmt.barEdges); },
    commentsForBar(i: number) {
      const barTimeStart = i / this.nSamples * this.duration;
      const barTimeEnd = (i + 1) / this.nSamples * this.duration;
      return this.comments.filter((c: AudioComment) =>
        hasOverlap([c.timeStart, c.timeEnd], [barTimeStart, barTimeEnd])
      ).sort((x: AudioComment, y: AudioComment) => x.rank - y.rank);
    },
    
    // Check if a comment is a parent of the currently active comment
    isActiveParentComment(cmt: AudioComment): boolean {
      if (!this.activeComment) return false;
      return cmt !== this.activeComment &&
             cmt.timeStart <= this.activeComment.timeStart &&
             cmt.timeEnd >= this.activeComment.timeEnd;
    },
    commentForBar(i: number) {
      const cmts = this.commentsForBar(i).sort(
        (x: AudioComment, y: AudioComment) => x.timeStart - y.timeStart
      );
      return cmts.length >= 1 ? cmts[cmts.length - 1] : null;
    },
    highlightComment(cmt: AudioComment) {
      this.highlightedComment = cmt;
      const commentEl = this.$refs.audiocomment[cmt.index].$el;
      commentEl.scrollIntoView(
        { block: 'nearest', inline: 'start', behavior: 'smooth' }
      );
    },
    highlightCommentForBar(i: number) {
      const cmt = this.commentForBar(i);
      cmt ? this.highlightComment(cmt) : this.highlightedComment = null;
    },
    unhighlightComment() { this.highlightedComment = null; },

    highlightBars(ixs: number[]) { this.highlightedBars = ixs; },
    unhighlightBars() { this.highlightedBars = []; },

    getBarHighlightHeight(s: number, rank: number, cmts: Array<AudioComment>) {
      // A bunch of magic numbers to get reasonable-looking
      // px values for CSS 'height' property
      const maxRank = Math.max(...cmts.map(x => x.rank));
      const maxRankGlobal = Math.max(
        ...this.comments.map((x: AudioComment) => x.rank)
      );
      const val = (Math.max(...this.filteredData) - s)
      const scaling = 50;
      const rankScaling = 3;
      const padding = 7 + rankScaling * maxRankGlobal;
      if (rank < maxRank) {
        const nextRank = cmts.filter(x => x.rank > rank)[0].rank;
        return rankScaling * (nextRank - rank);
      }
      return val * scaling + padding - rankScaling * rank;
    },
    getBarHighlightMarginTop(
      s: number, rank: number, cmt: AudioComment, cmts: Array<AudioComment>
    ) {
      // Arcane incantation to calculate CSS 'margin-top' property
      const height = this.getBarHighlightHeight(s, rank, cmts);
      const allRanks = cmts.map(x => x.rank);
      if (
        rank == 0 && cmts.length == 1
        || rank > cmts.indexOf(cmt) && rank != Math.max(...allRanks)
      )
        return -height;
      if (rank == Math.min(...allRanks))
        return -this.getBarHighlightHeight(s, rank, cmts.slice(0, -rank));
      return 0;
    },

    hasBeginSeparator(cmt: AudioComment, i: number) {
      // Whether the first waveform bin of the comment's time window
      // should be visually separated (because of overlap with the
      // previous comment
      if (i == 0) return false;  // first bin/bar has no overlap
      const cmtsPrevBar = this.commentsForBar(i - 1);
      if (!cmtsPrevBar || cmtsPrevBar.length == 0) return false;
      const areBarsAdjacent = this.startBars.includes(i)
        && this.endBars.includes(i - 1)
        && ! cmtsPrevBar.includes(cmt);
      const maxRankPrevBar = Math.max(...cmtsPrevBar.map((x: AudioComment) => x.rank));
      const isOverlapBegin = cmt.rank > 0 && cmt.rank > maxRankPrevBar;
      return areBarsAdjacent || isOverlapBegin;
    },
    hasEndSeparator(cmt: AudioComment, i: number) {
      // Whether the last waveform bin of the comment's time window
      // should be visually separated (because of overlap with the
      // next comment)
      if (i > this.filteredData.length - 1) return false;
      const cmtsNextBar = this.commentsForBar(i + 1);
      if (!cmtsNextBar || cmtsNextBar.length == 0) return false;
      const isOverlapEnd = cmt.rank > 0
        && cmt.rank > cmtsNextBar[cmtsNextBar.length - 1].rank;
      return this.endBars.includes(i) && isOverlapEnd;
    },

    // Video-related methods
    toggleVideoEmbed() {
      this.showVideoEmbed = !this.showVideoEmbed;
      this.updateVideoToggleIcon();
      
      // Sync video with current audio state when showing
      if (this.showVideoEmbed && this.$refs.videoElement) {
        this.$nextTick(() => {
          const video = this.$refs.videoElement as HTMLVideoElement;
          video.currentTime = this.currentTime;
          video.loop = this.looping;
          if (this.playing) {
            video.play();
          }
        });
      }
    },
    updateVideoToggleIcon() {
      if (this.$refs.videoToggleIcon) {
        setIcon(this.$refs.videoToggleIcon, this.showVideoEmbed ? 'eye-off' : 'eye');
      }
    },
    videoTimeUpdateHandler() {
      if (this.showVideoEmbed && this.$refs.videoElement) {
        const video = this.$refs.videoElement as HTMLVideoElement;
        // Only update currentTime from video - don't sync audio here to avoid lag
        this.currentTime = video.currentTime;
      }
    },
    videoLoadedHandler() {
      if (this.$refs.videoElement) {
        const video = this.$refs.videoElement as HTMLVideoElement;
        if (!this.duration) {
          this.duration = video.duration;
        }
      }
    },
    videoPlayHandler() {
      if (this.showVideoEmbed) {
        // Sync audio playback
        if (!this.isCurrent()) {
          this.audio.src = this.srcPath;
        }
        const video = this.$refs.videoElement as HTMLVideoElement;
        this.audio.currentTime = video.currentTime;
        this.audio.play();
        this.playing = true;
        this.setBtnIcon('pause');
        this.updateVideoControlIcons();
        
        // Start smooth time updates
        this.startSmoothTimeUpdate();
      }
    },
    videoPauseHandler() {
      if (this.showVideoEmbed && !this.audio.paused) {
        this.audio.pause();
        this.playing = false;
        this.setBtnIcon('play');
        this.updateVideoControlIcons();
        
        // Stop smooth time updates
        this.stopSmoothTimeUpdate();
      }
    },
    syncVideoWithAudio() {
      if (this.showVideoEmbed && this.$refs.videoElement) {
        const video = this.$refs.videoElement as HTMLVideoElement;
        if (Math.abs(video.currentTime - this.audio.currentTime) > 0.5) {
          video.currentTime = this.audio.currentTime;
        }
        if (this.playing && video.paused) {
          video.play();
        } else if (!this.playing && !video.paused) {
          video.pause();
        }
      }
    },

    copyTimestampToClipboard() {
      navigator.clipboard.writeText(this.displayedCurrentTime);
    },
    setCopyTimestampTooltip() {
      const elem = this.$refs.currentTime;
      setTooltip(elem, "Copy timestamp", { 'delay': 150 });
    },
    setWvTimestampTooltip(i: number) {
      // Legacy method kept for compatibility - no longer used for bar-by-bar tooltips
      const time = i / this.nSamples * this.duration;
      return secondsToString(time);
    },
    
    // Calculate precise time from mouse X position within waveform
    getTimeFromMouseEvent(event: MouseEvent): number {
      const waveformEl = this.$refs.waveformAudio as HTMLElement;
      if (!waveformEl || this.duration <= 0) return 0;
      
      const rect = waveformEl.getBoundingClientRect();
      const x = event.clientX - rect.left;
      const ratio = Math.max(0, Math.min(1, x / rect.width));
      return ratio * this.duration;
    },
    
    // Get bar index from mouse position
    getBarFromMouseEvent(event: MouseEvent): number {
      const time = this.getTimeFromMouseEvent(event);
      return this.barForTime(time);
    },
    
    // Handle waveform mousemove for realistic time display and scrubbing
    handleWaveformMouseMove(event: MouseEvent) {
      const time = this.getTimeFromMouseEvent(event);
      const barIndex = this.barForTime(time);
      
      // Highlight comment for this bar
      this.highlightCommentForBar(barIndex);
      
      // Show/update tooltip with precise time
      this.showWaveformTooltip(event, time);
      
      // If scrubbing, update playhead position
      if (this.isScrubbing) {
        this.scrubToTime(time);
      }
    },
    
    // Handle waveform mousedown - start scrubbing
    handleWaveformMouseDown(event: MouseEvent) {
      // Prevent text selection while scrubbing
      event.preventDefault();
      
      const time = this.getTimeFromMouseEvent(event);
      
      // Start scrubbing
      this.isScrubbing = true;
      this.wasPlayingBeforeScrub = this.playing;
      
      // Optionally pause during scrub for smoother experience
      if (this.playing) {
        this.pause();
      }
      
      // Set initial position
      this.scrubToTime(time);
      
      // Add document-level listeners for mouseup and mousemove outside waveform
      document.addEventListener('mouseup', this.handleDocumentMouseUp);
      document.addEventListener('mousemove', this.handleDocumentMouseMove);
    },
    
    // Handle document-level mousemove during scrubbing (for dragging outside waveform)
    handleDocumentMouseMove(event: MouseEvent) {
      if (!this.isScrubbing) return;
      
      const waveformEl = this.$refs.waveformAudio as HTMLElement;
      if (!waveformEl || this.duration <= 0) return;
      
      const rect = waveformEl.getBoundingClientRect();
      const x = event.clientX - rect.left;
      // Clamp to valid range even if mouse is outside waveform
      const ratio = Math.max(0, Math.min(1, x / rect.width));
      const time = ratio * this.duration;
      
      this.scrubToTime(time);
      this.showWaveformTooltip(event, time);
    },
    
    // Handle document-level mouseup - stop scrubbing
    handleDocumentMouseUp(event: MouseEvent) {
      if (!this.isScrubbing) return;
      
      this.isScrubbing = false;
      
      // Remove document-level listeners
      document.removeEventListener('mouseup', this.handleDocumentMouseUp);
      document.removeEventListener('mousemove', this.handleDocumentMouseMove);
      
      // Resume playback if was playing before scrub
      if (this.wasPlayingBeforeScrub) {
        this.play();
      }
      
      this.wasPlayingBeforeScrub = false;
    },
    
    // Scrub to a specific time (update currentTime without triggering audio timeupdate)
    scrubToTime(time: number) {
      // Clamp time to valid range
      const clampedTime = Math.max(0, Math.min(this.duration, time));
      
      this.currentTime = clampedTime;
      
      // Update audio element
      if (!this.isCurrent()) {
        this.audio.src = this.srcPath;
      }
      this.audio.currentTime = clampedTime;
      
      // Sync video if showing
      if (this.showVideoEmbed && this.$refs.videoElement) {
        (this.$refs.videoElement as HTMLVideoElement).currentTime = clampedTime;
      }
    },
    
    // Show tooltip at mouse position
    showWaveformTooltip(event: MouseEvent, time: number) {
      // Create tooltip element if it doesn't exist
      if (!this.waveformTooltipEl) {
        this.waveformTooltipEl = document.createElement('div');
        this.waveformTooltipEl.className = 'waveform-time-tooltip';
        document.body.appendChild(this.waveformTooltipEl);
      }
      
      // Update tooltip content and position
      this.waveformTooltipEl.textContent = secondsToString(time);
      this.waveformTooltipEl.style.display = 'block';
      this.waveformTooltipEl.style.left = `${event.clientX}px`;
      this.waveformTooltipEl.style.top = `${event.clientY - 35}px`;
    },
    
    // Hide waveform tooltip
    hideWaveformTooltip() {
      if (this.waveformTooltipEl) {
        this.waveformTooltipEl.style.display = 'none';
      }
    },
    
    // Start smooth time update using requestAnimationFrame for real-time display
    startSmoothTimeUpdate() {
      // Cancel any existing animation frame
      this.stopSmoothTimeUpdate();
      
      const updateLoop = () => {
        if (this.playing && this.isCurrent() && !this.audio.paused) {
          // Directly poll audio.currentTime for smooth updates
          this.currentTime = this.audio.currentTime;
          this.animationFrameId = requestAnimationFrame(updateLoop);
        }
      };
      
      this.animationFrameId = requestAnimationFrame(updateLoop);
    },
    
    // Stop smooth time update
    stopSmoothTimeUpdate() {
      if (this.animationFrameId !== null) {
        cancelAnimationFrame(this.animationFrameId);
        this.animationFrameId = null;
      }
    },
    
    // Handle keyboard events - spacebar to toggle play/pause
    handleKeydown(event: KeyboardEvent) {
      // Only handle if player is focused
      if (event.code === 'Space' || event.key === ' ') {
        // Prevent default scrolling behavior
        event.preventDefault();
        event.stopPropagation();
        
        // Toggle play/pause
        this.togglePlay();
      }
    },
    
    // Handle click inside player - focus for keyboard shortcuts
    handlePlayerClick(event: MouseEvent) {
      // Focus with preventScroll to avoid scroll jumping
      this.$el.focus({ preventScroll: true });
    },
    
    
    // Handle mouse enter player - focus for keyboard shortcuts (hover enables spacebar control)
    handlePlayerMouseEnter() {
      this.isPlayerHovered = true;
      // Auto-focus the player so keyboard shortcuts work immediately on hover
      this.$el.focus({ preventScroll: true });
    },
    
    // Track when mouse leaves player area - blur to allow normal scrolling
    handlePlayerMouseLeave() {
      this.isPlayerHovered = false;
      // Blur the element so spacebar scrolls normally outside
      this.$el.blur();
    }
  },

  created() {
    this.loadFile();
  },
  mounted() {
    this.button = this.$refs.playpause as HTMLSpanElement;
    this.setBtnIcon('play');
    setIcon(this.$refs.loopButton, 'repeat');
    setIcon(this.$refs.skipBackButton, 'skip-back');
    
    // Initialize video toggle icon if video
    if (this.isVideo) {
      this.updateVideoToggleIcon();
    }

    // Add event listeners
    document.addEventListener('allpause', () => {
      this.setBtnIcon('play');
      // Pause video if showing
      if (this.showVideoEmbed && this.$refs.videoElement) {
        (this.$refs.videoElement as HTMLVideoElement).pause();
      }
    });
    document.addEventListener('allresume', () => {
      if (this.isCurrent()) {
        this.setBtnIcon('pause');
        // Resume video if showing
        if (this.showVideoEmbed && this.$refs.videoElement) {
          (this.$refs.videoElement as HTMLVideoElement).play();
        }
      }
    })
    document.addEventListener('looptoggle', () => {
      if (this.isCurrent())
        this.toggleLooping();
    })
    this.audio.addEventListener('ended', () => {
      if (this.audio.src === this.srcPath)
        this.setBtnIcon('play');
    });
    this.$el.addEventListener('seek-to-timestamp', (event: CustomEvent) => {
      if (this.getSectionInfo()) {
        const { lineStart, lineEnd, timeInSeconds } = event.detail;
        if (
          timeInSeconds && timeInSeconds <= this.duration &&
          (lineStart == null && lineEnd == null ||
          lineStart == this.getSectionInfo().lineStart && lineEnd == this.getSectionInfo().lineEnd)
        ) {
          this.setPlayheadSecs(timeInSeconds);
          localStorage.removeItem('musicPlayerSeek');
        }
      }
    });

    this.$el.addEventListener('resize', () => {
      console.log(this.$el.clientWidth);
    })

    // Get current time
    if (this.audio.src === this.srcPath) {
      this.currentTime = this.audio.currentTime
      this.audio.addEventListener('timeupdate', this.timeUpdateHandler);
      this.setBtnIcon(this.audio.paused ? 'play' : 'pause');
      // Get current looping state
      this.looping = this.audio.loop;
      // Start smooth time updates if already playing
      if (!this.audio.paused) {
        this.playing = true;
        this.startSmoothTimeUpdate();
      }
    }

    // Load comments
    setTimeout(() => { this.comments = this.getComments(); });

    // Seek to timestamp based on cookie set after clicking a
    // timestamp link (as alternative to handling seek event)
    const seekToSeconds = localStorage.getItem('musicPlayerSeek');
    if (seekToSeconds && this.getSectionInfo()) {
      const [ lineStart, lineEnd, secondsStr ] = seekToSeconds.split(':');
      const seconds = parseFloat(secondsStr);
      if (
        seconds && seconds <= this.duration &&
        (lineStart == '' && lineEnd == '' ||
        lineStart == this.getSectionInfo().lineStart && lineEnd == this.getSectionInfo().lineEnd)
      ) {
        this.setPlayheadSecs(seconds);
        localStorage.removeItem('musicPlayerSeek');
      }
    }

    this.ro = new ResizeObserver(this.onResize);
    this.ro.observe(this.$el);
  },
  beforeUnmount() {
    this.audio.removeEventListener("timeupdate", this.timeUpdateHandler);
    // Stop smooth time updates
    this.stopSmoothTimeUpdate();
    // Clean up scrubbing listeners if still active
    document.removeEventListener('mouseup', this.handleDocumentMouseUp);
    document.removeEventListener('mousemove', this.handleDocumentMouseMove);
  },
  beforeDestroy() {
    this.ro.unobserve(this.$el);
    // Clean up tooltip element
    if (this.waveformTooltipEl) {
      this.waveformTooltipEl.remove();
      this.waveformTooltipEl = null;
    }
    // Clean up scrubbing listeners if still active
    document.removeEventListener('mouseup', this.handleDocumentMouseUp);
    document.removeEventListener('mousemove', this.handleDocumentMouseMove);
  }
})

</script>