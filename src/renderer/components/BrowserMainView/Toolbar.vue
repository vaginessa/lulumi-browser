<template lang="pug">
  #toolbar
    webview(src="https://tw.yahoo.com")
    webview(src="https://www.gamer.com.tw")
</template>

<style lang="less" scoped>
  #toolbar {
    display: flex;
    flex-direction: column;
    font-size: 15px;
    font-weight: 100;
    border-bottom: 1px solid #aaa;

    webview {
      height: 35px;
      width: 100vw;
    }
  }
</style>
