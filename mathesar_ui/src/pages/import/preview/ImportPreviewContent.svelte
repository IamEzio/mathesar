<script lang="ts">
  import { router } from 'tinro';

  import {
    CancelOrProceedButtonPair,
    Spinner,
  } from '@mathesar-component-library';
  import type { Database, SchemaEntry } from '@mathesar/AppTypes';
  import { columnsApi } from '@mathesar/api/columns';
  import type { DataFile } from '@mathesar/api/types/dataFiles';
  import type { TableEntry } from '@mathesar/api/types/tables';
  import type { Column } from '@mathesar/api/types/tables/columns';
  import {
    Field,
    FieldLayout,
    makeForm,
    requiredField,
    uniqueWith,
  } from '@mathesar/components/form';
  import InfoBox from '@mathesar/components/message-boxes/InfoBox.svelte';
  import { iconDeleteMajor } from '@mathesar/icons';
  import {
    getImportPreviewPageUrl,
    getSchemaPageUrl,
    getTablePageUrl,
  } from '@mathesar/routes/urls';
  import AsyncStore from '@mathesar/stores/AsyncStore';
  import { currentDbAbstractTypes } from '@mathesar/stores/abstract-types';
  import {
    generateTablePreview,
    getTypeSuggestionsForTable,
    patchTable,
    tables,
  } from '@mathesar/stores/tables';
  import { toast } from '@mathesar/stores/toast';
  import ColumnNamingStrategyInput from '../column-names/ColumnNamingStrategyInput.svelte';
  import ColumnTypeInferenceInput from '../inference/ColumnTypeInferenceInput.svelte';
  import ErrorInfo from './ErrorInfo.svelte';
  import ImportPreviewLayout from './ImportPreviewLayout.svelte';
  import ImportPreviewSheet from './ImportPreviewSheet.svelte';
  import {
    buildColumnPropertiesMap,
    finalizeColumns,
    getSkeletonRecords,
    makeDeleteTableRequest,
    makeHeaderUpdateRequest,
    processColumns,
  } from './importPreviewPageUtils';

  /** Set via back-end */
  const TRUNCATION_LIMIT = 20;

  const columnsFetch = new AsyncStore(columnsApi.list);
  const previewRequest = new AsyncStore(generateTablePreview);
  const typeSuggestionsRequest = new AsyncStore(getTypeSuggestionsForTable);
  const headerUpdate = makeHeaderUpdateRequest();
  const cancelationRequest = makeDeleteTableRequest();

  export let database: Database;
  export let schema: SchemaEntry;
  export let table: TableEntry;
  export let dataFile: DataFile;
  export let useColumnTypeInference = false;

  let columns: Column[] = [];
  let columnPropertiesMap = buildColumnPropertiesMap([]);

  $: otherTableNames = [...$tables.data.values()]
    .filter((t) => t.id !== table.id)
    .map((t) => t.name);
  $: customizedTableName = requiredField(table.name, [
    uniqueWith(otherTableNames, 'A table with this name already exists.'),
  ]);
  $: form = makeForm({ customizedTableName });
  $: records = $previewRequest.resolvedValue?.records ?? getSkeletonRecords();
  $: formInputsAreDisabled = !$previewRequest.isOk;
  $: canProceed = $previewRequest.isOk && $form.canSubmit;
  $: processedColumns = processColumns(columns, $currentDbAbstractTypes.data);

  async function init() {
    const columnsResponse = await columnsFetch.run(table.id);
    const fetchedColumns = columnsResponse?.resolvedValue?.results;
    if (!fetchedColumns) {
      return;
    }
    columns = fetchedColumns;
    columnPropertiesMap = buildColumnPropertiesMap(columns);
    if (useColumnTypeInference) {
      const response = await typeSuggestionsRequest.run(table.id);
      if (response.settlement?.state === 'resolved') {
        const typeSuggestions = response.settlement.value;
        columns = columns.map((column) => ({
          ...column,
          type: typeSuggestions[column.name] ?? column.type,
        }));
      }
    }
    await previewRequest.run({ table, columns });
  }
  $: table, useColumnTypeInference, void init();

  function reload(props: {
    table?: TableEntry;
    useColumnTypeInference?: boolean;
  }) {
    const tableId = props.table?.id ?? table.id;
    router.goto(
      getImportPreviewPageUrl(database.name, schema.id, tableId, {
        useColumnTypeInference:
          props.useColumnTypeInference ?? useColumnTypeInference,
      }),
      true,
    );
  }

  async function toggleHeader() {
    previewRequest.reset();
    const response = await headerUpdate.run({
      database,
      dataFile,
      firstRowIsHeader: !dataFile.header,
      schema,
      table,
      customizedTableName: $customizedTableName,
    });
    if (response.resolvedValue) {
      reload({ table: response.resolvedValue });
    }
  }

  async function toggleInference() {
    previewRequest.reset();
    reload({ useColumnTypeInference: !useColumnTypeInference });
  }

  function updateTypeRelatedOptions(updatedColumn: Column) {
    columns = columns.map((c) =>
      c.id === updatedColumn.id ? updatedColumn : c,
    );
    return previewRequest.run({ table, columns });
  }

  async function cancel() {
    const response = await cancelationRequest.run({ database, schema, table });
    if (response.isOk) {
      router.goto(getSchemaPageUrl(database.name, schema.id), true);
    } else {
      toast.fromError(response.error);
    }
  }

  async function finishImport() {
    try {
      await patchTable(table.id, {
        name: $customizedTableName,
        import_verified: true,
        columns: finalizeColumns(columns, columnPropertiesMap),
      });
      router.goto(getTablePageUrl(database.name, schema.id, table.id), true);
    } catch (err) {
      toast.fromError(err);
    }
  }
</script>

<ImportPreviewLayout>
  <Field field={customizedTableName} label="Table Name" />
  <FieldLayout>
    <ColumnNamingStrategyInput
      value={dataFile.header}
      on:change={toggleHeader}
      disabled={formInputsAreDisabled}
    />
  </FieldLayout>
  <FieldLayout>
    <ColumnTypeInferenceInput
      value={useColumnTypeInference}
      on:change={toggleInference}
      disabled={formInputsAreDisabled}
    />
  </FieldLayout>
  <FieldLayout>
    <InfoBox>
      You can customize column names and types within the preview below.
    </InfoBox>
  </FieldLayout>

  <svelte:fragment slot="preview">
    <h2 class="large-bold-header preview-header">Table Preview</h2>
    <div class="preview-content">
      {#if $columnsFetch.error}
        <ErrorInfo
          error={$columnsFetch.error}
          on:retry={init}
          on:delete={cancel}
        />
      {:else if !$previewRequest.hasInitialized}
        <div class="loading"><Spinner /></div>
      {:else if $previewRequest.error}
        <ErrorInfo
          error={$previewRequest.error}
          on:retry={init}
          on:delete={cancel}
        />
      {:else if $headerUpdate.error}
        <ErrorInfo
          error={$headerUpdate.error}
          on:retry={init}
          on:delete={cancel}
        />
      {:else}
        <div class="sheet-holder">
          <ImportPreviewSheet
            columns={processedColumns}
            isLoading={$previewRequest.isLoading}
            {updateTypeRelatedOptions}
            {columnPropertiesMap}
            {records}
          />
        </div>
        {#if records.length === TRUNCATION_LIMIT}
          <div class="truncation-alert">
            (Preview is truncated at {TRUNCATION_LIMIT} rows.)
          </div>
        {/if}
      {/if}
    </div>
  </svelte:fragment>

  <svelte:fragment slot="footer">
    {#if $cancelationRequest.isLoading}
      Cleaning up... <Spinner />
    {:else}
      <CancelOrProceedButtonPair
        onCancel={cancel}
        onProceed={finishImport}
        cancelButton={{ icon: iconDeleteMajor }}
        proceedButton={{ label: 'Confirm & create table' }}
        {canProceed}
      />
    {/if}
  </svelte:fragment>
</ImportPreviewLayout>

<style>
  .loading {
    text-align: center;
    font-size: 2rem;
    color: var(--slate-500);
  }
  .preview-header {
    margin: 0;
    padding: var(--size-small) var(--inset-page-section-padding);
    border-bottom: 1px solid var(--slate-200);
    border-top: solid 1px var(--slate-300);
    background: var(--white);
  }
  .preview-content {
    padding: var(--inset-page-section-padding);
  }
  .sheet-holder {
    max-width: fit-content;
    overflow: auto;
    margin: 0 auto;
    border: 1px solid var(--slate-200);
  }
  .truncation-alert {
    margin: 1rem auto 0 auto;
    max-width: max-content;
    color: var(--color-text-muted);
  }
</style>
