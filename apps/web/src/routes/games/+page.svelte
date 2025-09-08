<script lang="ts">
// 임시 게임 목록 데이터
const games = [
  { title: '샘플 게임', description: '등록된 게임이 여기에 표시됩니다.' }
];
</script>

<h1>게임 허브</h1>
<ul>
  {#each games as game}
    <li>{game.title} - {game.description}</li>
  {/each}
</ul>
