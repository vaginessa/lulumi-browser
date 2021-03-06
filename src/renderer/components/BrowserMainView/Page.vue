<template lang="pug">
  div
    transition(name="notification")
      #notification(v-show="showNotification && isActive")
        notification
    webview(:element-loading-text="$t('page.loading')", ref="webview", :class="isActive ? 'active' : 'hidden'")
    .findinpage-bar(ref="findinpageBar", v-show="!hidden && isActive")
      input(ref="findinpageInput", :placeholder="$t('page.findInPage.placeholder')")
      span(ref="findinpageCount")
      div
        i(ref="findinpagePreviousMatch", class="el-icon-arrow-up")
        i(ref="findinpageNextMatch", class="el-icon-arrow-down")
        i(ref="findinpageEnd", class="el-icon-circle-close")
</template>

<script>
  import urlUtil from '../../js/lib/url-util';

  import Event from '../../../api/extensions/event';

  import Notification from './Notification';

  export default {
    data() {
      return {
        hidden: true,
        requestId: null,
        showNotification: false,
        onMessageEvent: new Event(),
      };
    },
    props: [
      'isActive',
      'pageIndex',
    ],
    components: {
      Notification,
    },
    computed: {
      page() {
        return this.$store.getters.pages[this.pageIndex];
      },
      currentPageIndex() {
        return this.$store.getters.currentPageIndex;
      },
    },
    methods: {
      navigateTo(location) {
        if (this.$refs.webview) {
          this.$refs.webview.setAttribute('src', urlUtil.getUrlFromInput(location));
        }
      },
      webviewHandler(self, fnName) {
        return (event) => {
          if (self.$parent[fnName]) {
            self.$parent[fnName](event, this.pageIndex);
          }
        };
      },
    },
    watch: {
      isActive(newState) {
        if (newState && !this.hidden) {
          this.$nextTick(() => this.$refs.findinpageInput.focus());
        }
      },
    },
    mounted() {
      const webview = this.$refs.webview;
      const webviewEvents = {
        'load-commit': 'onLoadCommit',
        'did-start-loading': 'onDidStartLoading',
        'did-stop-loading': 'onDidStopLoading',
        'did-finish-load': 'onDidFinishLoading',
        'did-fail-load': 'onDidFailLoad',
        'did-get-response-details': 'onDidGetResponseDetails',
        'did-get-redirect-request': 'onDidGetRedirectRequest',
        'dom-ready': 'onDomReady',
        'page-title-set': 'onPageTitleSet',
        'ipc-message': 'onIpcMessage',
        'console-message': 'onConsoleMessage',
        'update-target-url': 'onUpdateTargetUrl',
        'media-started-playing': 'onMediaStartedPlaying',
        'media-paused': 'onMediaPaused',
        'page-favicon-updated': 'onPageFaviconUpdated',
        'enter-html-full-screen': 'onEnterHtmlFullScreen',
        'leave-html-full-screen': 'onLeaveHtmlFullScreen',
        'new-window': 'onNewWindow',
        'scroll-touch-begin': 'onScrollTouchBegin',
        'scroll-touch-end': 'onScrollTouchEnd',
        'context-menu': 'onContextMenu',
        'will-navigate': 'onWillNavigate',
        'did-navigate': 'onDidNavigate',
      };

      Object.keys(webviewEvents).forEach((key) => {
        webview.addEventListener(key, this.webviewHandler(this, webviewEvents[key]));
      });

      const ipc = this.$electron.ipcRenderer;

      const findinpage = {
        container: this.$refs.findinpageBar,
        input: this.$refs.findinpageInput,
        counter: this.$refs.findinpageCount,
        previous: this.$refs.findinpagePreviousMatch,
        next: this.$refs.findinpageNextMatch,
        endButton: this.$refs.findinpageEnd,
        activeWebview: webview,
        start: () => {
          findinpage.counter.textContent = '';
          this.hidden = false;
          this.$nextTick(() => {
            findinpage.input.focus();
            findinpage.input.select();
          });

          if (findinpage.input.value) {
            this.requestId = findinpage.activeWebview.findInPage(findinpage.input.value);
          }
        },
        end: () => {
          this.hidden = true;

          this.$nextTick(() => {
            if (findinpage.activeWebview) {
              findinpage.activeWebview.stopFindInPage('keepSelection');
              if (findinpage.input === document.activeElement) {
                findinpage.activeWebview.focus();
              }
            }
          });
        },
      };

      findinpage.endButton.addEventListener('click', () => {
        findinpage.end();
      });

      findinpage.input.addEventListener('input', (event) => {
        if (event.target.value) {
          this.requestId = findinpage.activeWebview.findInPage(event.target.value);
        }
      });

      findinpage.input.addEventListener('keypress', (event) => {
        if (event.keyCode === 13) {
          this.requestId = findinpage.activeWebview.findInPage(findinpage.input.value, {
            forward: true,
            findNext: true,
          });
        }
      });

      findinpage.previous.addEventListener('click', () => {
        if (findinpage.input.value) {
          this.requestId = findinpage.activeWebview.findInPage(findinpage.input.value, {
            forward: false,
            findNext: true,
          });
        }
      });

      findinpage.next.addEventListener('click', () => {
        if (findinpage.input.value) {
          this.requestId = findinpage.activeWebview.findInPage(findinpage.input.value, {
            forward: true,
            findNext: true,
          });
        }
      });

      webview.addEventListener('found-in-page', (event) => {
        if (event.result.requestId === this.requestId) {
          // for this.$tc pluralization
          let match;
          if (event.result.matches !== undefined) {
            if (event.result.matches === 0) {
              match = 1;
            } else {
              match = 2;
            }
            findinpage.counter.textContent = `
              ${this.$t('page.findInPage.status', { activeMatch: event.result.activeMatchOrdinal, matches: event.result.matches })} ${this.$tc('page.findInPage.match', match)}
            `;
          }
        }
      });

      // Every page would add a listenter to the startFindInPage event,
      // so ignore the listenters warning.
      ipc.setMaxListeners(0);
      ipc.on('startFindInPage', () => {
        if (this.pageIndex === this.$store.getters.currentPageIndex) {
          if (this.hidden) {
            findinpage.start();
            findinpage.counter.textContent = `
              ${this.$t('page.findInPage.status', { activeMatch: 0, matches: 0 })} ${this.$tc('page.findInPage.match', 1)}
            `;
          } else {
            findinpage.input.focus();
            findinpage.input.select();
          }
        }
      });

      const nav = this.$parent.$el.querySelector('#nav');
      webview.style.height
        = `calc(100vh - ${nav.clientHeight}px)`;
      this.$el.querySelector('.findinpage-bar').style.top = `${nav.clientHeight}px`;
      this.navigateTo(this.page.location);
    },
    beforeDestroy() {
      const ipc = this.$electron.ipcRenderer;
      ipc.removeAllListeners([
        'startFindInPage',
      ]);
    },
  };
</script>

<style lang="less" scoped>
  #notification {
    width: 100vw;
    height: 35px;
    background: rgba(255, 193, 7, 0.28);
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .notification-enter-active, .notification-leave-active {
    transition: opacity .5s;
  }
  .notification-enter, .notification-leave-active {
    opacity: 0
  }

  webview {
    height: 0px;
    width: 100vw;
    outline: none;
    position: relative;

    &[hidden] {
      flex: 0 1;
      width: 0px;
      height: 0px !important;
    }

    &.fullscreen {
      position: fixed;
      top: 0;
      left: 0;
      width: 100vw;
      height: 100vh;
      z-index: 99;
    }

    /* hack to work around display: none issues with webviews */
    &.hidden {
      flex: 0 1;
      width: 0px;
      height: 0px !important;
    }
  }

  .findinpage-bar {
    right: 0px;
    color: ghostwhite;
    background: rgba(105, 105, 105, 0.8);
    border-bottom-left-radius: 4px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    padding: 5px 10px;
    animation: slideIn 25ms;
    position: absolute;
    -webkit-user-select: none;

    *:nth-child(1) {
      flex: 1;
    }

    *:nth-child(2) {
      flex: 2;
    }

    div {
      flex: 1;
      i.el-icon-arrow-up, i.el-icon-arrow-down, i.el-icon-circle-close {
        border: 1px solid transparent;
        font-size: 16px;
        margin: 0 2px;
        opacity: 0.5;
      }
    }
  }
</style>
