<script lang="ts">
  import NameWithIcon from '@mathesar/components/NameWithIcon.svelte';
  import { iconUiTypeText } from '@mathesar/icons';
</script>

<NameWithIcon icon={iconUiTypeText} truncate={false}>
  Use "Text" for all column data types
  <em>(faster for large imports)</em>
</NameWithIcon>
