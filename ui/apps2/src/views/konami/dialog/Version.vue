<template>
  <div class="version">
    <div>Versions V3.0.27</div>
    <div class="updata">{{ $t('public.lastUpdate') }}: 2020-11-17 15:05</div>
  </div>
</template>

<script>
export default {
  data() {
    return {
    }
  }
}
</script>

<style lang="scss" scoped>
.version{
  padding: 40px;
  font-size: 28px;
  text-align: left;
  padding-left: 80px;
  .updata{
    margin-top: 10px;
  }
}
</style>
