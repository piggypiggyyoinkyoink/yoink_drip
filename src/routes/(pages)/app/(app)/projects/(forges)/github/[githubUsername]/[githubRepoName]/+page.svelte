<script lang="ts">
  import type { PageData } from './$types';
  import ProjectProfile from '../../../components/project-profile/project-profile.svelte';

  export let data: PageData;
</script>

<ProjectProfile
  repoExists={data.repoExists}
  project={data.project}
  description={data.description}
  newRepo={data.newRepo}
  correctCasingRepo={data.correctCasingRepo}
/>
