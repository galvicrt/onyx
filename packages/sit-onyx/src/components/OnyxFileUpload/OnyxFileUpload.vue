<script
  lang="ts"
  setup
  generic="
    TMultiple extends boolean = false,
    TModelValue extends (TMultiple extends true ? File[] : Nullable<File>) = TMultiple extends true
      ? File[]
      : Nullable<File>
  "
>
import { iconCloudArrowUp, iconTrash } from "@sit-onyx/icons";
import { computed, ref, useId, useTemplateRef, watch } from "vue";
import { useFileSize } from "../../composables/useFileSize.js";
import { useFormElementError } from "../../composables/useFormElementError.js";
import { SKELETON_INJECTED_SYMBOL } from "../../composables/useSkeletonState.js";
import { useVModel } from "../../composables/useVModel.js";
import { injectI18n } from "../../i18n/index.js";
import type { Nullable } from "../../types/utils.js";
import { mergeVueProps, useRootAttrs } from "../../utils/attrs.js";
import { userConsole } from "../../utils/console.js";
import { validateFileType } from "../../utils/file.js";
import { convertBinaryPrefixToBytes } from "../../utils/numbers.js";
import { asArray } from "../../utils/objects.js";
import { useForwardProps } from "../../utils/props.js"; // Standard Prop Forwarding Nutzen
import { OnyxFileUploadSVG } from "../illustrations/index.js";
import OnyxFileCard from "../OnyxFileCard/OnyxFileCard.vue";
import type { FileCardStatus, OnyxFileCardProps } from "../OnyxFileCard/types.js";
import { FORM_INJECTED_SYMBOL, useFormContext } from "../OnyxForm/OnyxForm.core.js";
import OnyxFormElementV2 from "../OnyxFormElementV2/OnyxFormElementV2.vue";
import { customMessageToFormElementV2Message } from "../OnyxFormElementV2/useLegacyFormElementProps.js";
import OnyxIcon from "../OnyxIcon/OnyxIcon.vue";
import OnyxIconButton from "../OnyxIconButton/OnyxIconButton.vue";
import OnyxSystemButton from "../OnyxSystemButton/OnyxSystemButton.vue";
import OnyxVisuallyHidden from "../OnyxVisuallyHidden/OnyxVisuallyHidden.vue";
import type { MediaType, OnyxFileUploadProps } from "./types.js";

defineOptions({ inheritAttrs: false });

const props: OnyxFileUploadProps<TMultiple> = withDefaults(
  defineProps<OnyxFileUploadProps<TMultiple>>(),
  {
    accept: () => [],
    size: "large",
    listType: "list",
    disabled: FORM_INJECTED_SYMBOL,
    skeleton: SKELETON_INJECTED_SYMBOL,
    showError: FORM_INJECTED_SYMBOL,
    requiredMarker: FORM_INJECTED_SYMBOL,
    reserveMessageSpace: FORM_INJECTED_SYMBOL,
  },
);

const emit = defineEmits<{
  /**
   * Emitted when the validity state of the file input changes.
   */
  validityChange: [validity: ValidityState];
  /**
   * Emitted when the file input changes
   */
  "update:modelValue": [value: TModelValue];
}>();

defineSlots<{
  /**
   * Optional slot to override the displayed file card for each file.
   *
   * @param file The current file being rendered
   * @param props File card props that would originally be passed to the OnyxFileCard if this slot
   *   would not be used
   */
  default?(props: { file: File; props: OnyxFileCardProps }): unknown;
}>();

const { disabled } = useFormContext(props);
const { t } = injectI18n();
const { restAttrs, rootAttrs } = useRootAttrs();
const { formatFileSize } = useFileSize();

const modelValue = useVModel<typeof props, "modelValue", TModelValue>({
  props,
  emit,
  key: "modelValue",
});

const input = useTemplateRef<HTMLInputElement>("input");

const currentFiles = computed<File[]>(() => {
  const files = asArray<Nullable<File>>(modelValue.value ?? []);
  return files.filter((file) => file != null);
});

const requiredError = computed(() => {
  if (props.required && currentFiles.value.length === 0) {
    return t.value("fileUpload.requiredError");
  }

  return undefined;
});

const { vCustomValidity, errorMessages } = useFormElementError({
  props,
  emit,
  error: requiredError,
});

const formElementV2Props = useForwardProps(props, OnyxFormElementV2);

const label = computed(
  () => props.label ?? { label: t.value("fileUpload.clickToUpload"), hidden: true },
);
const error = computed(
  () => props.error ?? customMessageToFormElementV2Message(errorMessages.value),
);

const hideFiles = ref(false);

/**
 * Files larger than this are not hashed for duplicate detection to keep memory usage low.
 */
const MAX_DUPLICATE_CHECK_SIZE = 50 * 1024 * 1024;

/**
 * Max. time in milliseconds to wait for a file import from `importUrl`.
 */
const IMPORT_TIMEOUT = 30_000;

/**
 * Files whose content is identical to another selected file.
 */
const duplicateFiles = ref(new Set<File>());

/**
 * Creates a SHA-256 hash of the file content so only the hash (and not the whole file) needs to be
 * kept in memory.
 */
const hashFile = async (file: File) => {
  const digest = await crypto.subtle.digest("SHA-256", await file.arrayBuffer());
  return Array.from(new Uint8Array(digest), (byte) => byte.toString(16).padStart(2, "0")).join("");
};

watch(
  currentFiles,
  async (files, _, onCleanup) => {
    let isStale = false;
    onCleanup(() => (isStale = true));

    // only files with the same size can have identical content
    const filesBySize = new Map<number, File[]>();
    files.forEach((file) => {
      if (file.size > MAX_DUPLICATE_CHECK_SIZE) return;
      if (props.maxSize && file.size > convertBinaryPrefixToBytes(props.maxSize)) return;
      filesBySize.set(file.size, [...(filesBySize.get(file.size) ?? []), file]);
    });
    const candidates = [...filesBySize.values()].filter((group) => group.length > 1).flat();

    // hashing requires a secure context, so duplicate detection is skipped otherwise
    if (!candidates.length || !globalThis.crypto?.subtle) {
      duplicateFiles.value = new Set();
      return;
    }

    // files are read one after another so only a single file is loaded into memory at a time
    const filesByHash = new Map<string, File[]>();
    for (const file of candidates) {
      const hash = await hashFile(file);
      if (isStale) return;
      filesByHash.set(hash, [...(filesByHash.get(hash) ?? []), file]);
    }

    duplicateFiles.value = new Set(
      [...filesByHash.values()].filter((group) => group.length > 1).flat(),
    );
  },
  { immediate: true },
);

const formattedHint = computed(() => {
  if (!props.hint) return undefined;
  const lastFile = currentFiles.value.at(-1);
  return props.hint.replaceAll("{filename}", lastFile?.name ?? "");
});

const hintId = useId();
const isImporting = ref(false);
const importError = ref(false);

const getFilenameFromUrl = (url: URL) => {
  const filename = url.pathname.split("/").at(-1);
  if (!filename) return "download";
  try {
    return decodeURIComponent(filename);
  } catch {
    return filename;
  }
};

/**
 * Reads the filename from a `Content-Disposition` header, e.g. `attachment; filename="report.pdf"`.
 * Path segments are stripped since the header value is untrusted.
 */
const getFilenameFromHeader = (header: string | null) => {
  const match = header?.match(/filename\*?=(?:UTF-8'')?"?([^";]+)"?/i);
  if (!match?.[1]) return undefined;
  let filename = match[1];
  try {
    filename = decodeURIComponent(filename);
  } catch {
    // keep the raw value
  }
  return filename.split(/[\\/]/).at(-1)?.trim() || undefined;
};

/**
 * Downloads the file from the configured `importUrl` and adds it to the selection.
 */
const importFromUrl = async () => {
  if (!props.importUrl) return;
  isImporting.value = true;
  importError.value = false;

  try {
    const url = new URL(props.importUrl, window.location.href);
    if (url.protocol !== "http:" && url.protocol !== "https:") {
      throw new Error(`Unsupported protocol "${url.protocol}"`);
    }

    const response = await fetch(url, {
      credentials: "same-origin",
      // applies to both the request and reading the response body
      signal: AbortSignal.timeout(IMPORT_TIMEOUT),
    });
    if (!response.ok) throw new Error(`Request failed with status ${response.status}`);

    const blob = await response.blob();
    // the component might have been disabled while the file was downloading
    if (disabled.value) return;

    // use the final URL so redirects (e.g. "/download" -> "/report.pdf") result in the correct filename
    const filename =
      getFilenameFromHeader(response.headers.get("Content-Disposition")) ??
      getFilenameFromUrl(response.url ? new URL(response.url) : url);
    setFiles([new File([blob], filename, { type: blob.type })]);
  } catch (error) {
    importError.value = true;
    userConsole?.error("Error while importing file from URL", props.importUrl, error);
  } finally {
    isImporting.value = false;
  }
};

const fileStatuses = computed((): (FileCardStatus | undefined)[] => {
  return currentFiles.value.map((file, index) => {
    if (props.accept && !validateFileType(file, props.accept)) {
      return {
        text: t.value("fileUpload.status.fileTypeError", {
          extension: file.name.split(".").at(-1),
        }),
        color: "danger",
      };
    }
    if (props.maxSize && file.size > convertBinaryPrefixToBytes(props.maxSize)) {
      return {
        text: t.value("fileUpload.status.fileSizeError", {
          size: formatFileSize.value(props.maxSize),
        }),
        color: "danger",
      };
    }
    const totalSize = currentFiles.value.slice(0, index + 1).reduce((sum, f) => sum + f.size, 0);
    if (props.maxTotalSize && totalSize > convertBinaryPrefixToBytes(props.maxTotalSize)) {
      return {
        text: t.value("fileUpload.status.maxFileSizeError", {
          size: formatFileSize.value(props.maxTotalSize),
        }),
        color: "danger",
      };
    }
    if (props.multiple && props.maxCount && index >= props.maxCount) {
      return {
        text: t.value("fileUpload.status.maxCountError", { count: props.maxCount }),
        color: "danger",
      };
    }
    if (duplicateFiles.value.has(file)) {
      return { text: t.value("fileUpload.status.duplicateWarning"), color: "warning" };
    }
  });
});

const currentFileProps = computed(() => {
  return currentFiles.value.map((file, index) => ({
    file,
    props: {
      filename: file.name,
      size: file.size,
      type: file.type as MediaType,
      link: createFileURL(file),
      status: fileStatuses.value[index],
    } satisfies OnyxFileCardProps,
  }));
});

/**
 * Sets the currently selected files by considering all relevant props (e.g. replace etc.).
 */
const setFiles = (files: File[]) => {
  let newFiles: File | File[] | undefined;

  if (!props.multiple) {
    newFiles = files[0];
  } else if (props.replace) {
    newFiles = files.slice();
  } else {
    newFiles = currentFiles.value.concat(files);
  }

  modelValue.value = newFiles as TModelValue;
};

const removeFile = (fileToRemove: File) => {
  if (props.multiple) {
    const newFiles = currentFiles.value.filter((file) => file !== fileToRemove);
    modelValue.value = newFiles as TModelValue;
  } else {
    modelValue.value = undefined as TModelValue;
  }
};

const handleChange = (event: Event) => {
  const target = event.target as HTMLInputElement;
  const files = Array.from(target.files ?? []);
  setFiles(files);
  target.value = "";
};

const showDetails = computed(() => {
  return (
    props.size === "large" ||
    ((props.maxSize || props.maxTotalSize || props.maxCount || props.accept?.length) &&
      props.size !== "small")
  );
});

const isDragging = ref(false);

const handleDrop = (event: DragEvent) => {
  if (disabled.value) return;
  isDragging.value = false;

  const files = Array.from(event.dataTransfer?.files ?? []);
  setFiles(files);
};

const handleDragEnter = () => {
  if (disabled.value) return;
  isDragging.value = true;
};

const createFileURL = (file: File) => {
  try {
    return URL.createObjectURL(file);
  } catch (error) {
    userConsole?.error("Error while creating URL for file", file, error);
    return undefined;
  }
};

const shouldShowFileList = computed(() => {
  if (props.listType === "hidden" || !currentFiles.value.length) return false;
  if (props.listType === "button") return !hideFiles.value;
  return true;
});
</script>

<template>
  <OnyxFormElementV2
    v-bind="mergeVueProps(formElementV2Props, rootAttrs)"
    :class="['onyx-file-upload-wrapper', `onyx-file-upload-wrapper--${props.size}`]"
    :label
    :error
    unstyled
  >
    <template #default="inputProps">
      <div class="onyx-file-upload-container">
        <button
          type="button"
          :class="[
            'onyx-file-upload',
            `onyx-file-upload--${props.size}`,
            { 'onyx-file-upload--dragging': isDragging },
          ]"
          :disabled="disabled"
          @dragenter="handleDragEnter"
          @dragleave="isDragging = false"
          @dragover.prevent
          @drop.prevent="handleDrop"
          :aria-describedby="formattedHint ? hintId : undefined"
          @click="input?.click()"
        >
          <OnyxFileUploadSVG
            v-if="props.size === 'large'"
            class="onyx-file-upload__default-illustration"
            :disabled="disabled"
            :active="isDragging"
          />
          <OnyxFileUploadSVG
            v-if="props.size === 'large'"
            class="onyx-file-upload__error-illustration"
            error
            :disabled="disabled"
            :active="isDragging"
          />
          <div v-else class="onyx-file-upload__icon">
            <OnyxIcon :icon="iconCloudArrowUp" />
            <span> {{ t("fileUpload.select") }}</span>
          </div>

          <div v-if="showDetails" class="onyx-file-upload__content">
            <p v-if="props.size === 'large'" class="onyx-file-upload__label onyx-text">
              <u>{{ t("fileUpload.clickToUpload") }}</u> {{ t("fileUpload.orDragAndDrop") }}.
            </p>

            <p
              v-if="props.maxSize || props.maxTotalSize"
              class="onyx-file-upload__text onyx-text--small"
            >
              {{ t("fileUpload.maxFileSize") }}:

              <template v-if="props.maxSize && props.maxTotalSize">
                {{ formatFileSize(props.maxSize) }} ({{ formatFileSize(props.maxTotalSize) }}
                {{ t("fileUpload.inTotal") }})
              </template>

              <template v-else-if="props.maxSize">
                {{ formatFileSize(props.maxSize) }}
              </template>
              <template v-else-if="props.maxTotalSize">
                {{ formatFileSize(props.maxTotalSize) }} {{ t("fileUpload.inTotal") }}
              </template>
            </p>

            <p
              v-if="props.multiple && props.maxCount"
              class="onyx-file-upload__text onyx-text--small"
            >
              {{ t("fileUpload.maxFileCount", { n: props.maxCount }) }}
            </p>

            <p v-if="props.accept?.length" class="onyx-file-upload__text onyx-text--small">
              {{ t("fileUpload.allowedFileTypes", { types: props.accept.join(", ") }) }}
            </p>
          </div>
        </button>

        <OnyxVisuallyHidden class="onyx-file-upload__visually-hidden-input">
          <input
            ref="input"
            v-custom-validity
            v-bind="
              mergeVueProps(restAttrs, {
                ...inputProps,
                // we must NOT define required on the input here since this does not work
                // with our validation and will make the input invalid even if files are selected
                required: undefined,
              })
            "
            aria-hidden="true"
            tabindex="-1"
            class="onyx-file-upload__input"
            type="file"
            :accept="props.accept?.length ? props.accept.join(',') : undefined"
            :multiple="props.multiple"
            :name="props.name"
            :disabled="disabled"
            @change="handleChange"
          />
        </OnyxVisuallyHidden>

        <OnyxSystemButton
          v-if="props.importUrl"
          class="onyx-file-upload__import-button"
          :label="t('fileUpload.importFromUrl')"
          :disabled="disabled || isImporting"
          @click="importFromUrl"
        />

        <OnyxSystemButton
          v-if="props.listType === 'button' && currentFiles.length"
          class="onyx-file-upload__list-button"
          :label="hideFiles ? t('fileUpload.revealFilesButton') : t('fileUpload.hideFilesButton')"
          @click="hideFiles = !hideFiles"
        />

        <div
          v-if="shouldShowFileList"
          :class="[
            'onyx-file-upload__list',
            { 'onyx-file-upload__list--max-height': props.listType === 'maxHeight' },
          ]"
        >
          <template v-for="fileProps in currentFileProps" :key="fileProps.props.filename">
            <slot :file="fileProps.file" :props="fileProps.props">
              <OnyxFileCard v-bind="fileProps.props">
                <template #actions>
                  <OnyxIconButton
                    color="danger"
                    :icon="iconTrash"
                    :label="t('fileUpload.removeFile')"
                    :disabled="disabled"
                    @click="removeFile(fileProps.file)"
                  />
                </template>
              </OnyxFileCard>
            </slot>
          </template>
        </div>

        <p v-if="importError" class="onyx-file-upload__text onyx-text--small" role="alert">
          {{ t("fileUpload.importError") }}
        </p>

        <p v-if="formattedHint" :id="hintId" class="onyx-file-upload__text onyx-text--small">
          {{ formattedHint }}
        </p>
      </div>
    </template>
  </OnyxFormElementV2>
</template>

<style lang="scss">
@use "../../styles/mixins/layers.scss";
@use "../../styles/mixins/input.scss";
@use "../OnyxFileCard/OnyxFileCard.scss";

@include layers.component() {
  .onyx-file-upload-wrapper {
    --onyx-file-upload-max-files: 3;

    --onyx-file-upload-border-color: var(--onyx-color-component-border-neutral);
    --onyx-file-upload-active-border-color: var(--onyx-color-component-border-neutral-hover);
    --onyx-file-upload-error-illustration-display: none;
    --onyx-file-upload-default-illustration-display: block;
    --onyx-file-upload-outline-color: var(--onyx-color-component-focus-primary);
    --onyx-file-upload-active-background-color: var(--onyx-color-base-primary-100);
    --onyx-file-upload-active-label-color: var(--onyx-color-text-icons-primary-intense);
    --onyx-file-upload-hover-svg-background-color-bubble: var(--onyx-color-base-primary-800);

    --onyx-file-upload-small-hover-background-color: var(--onyx-background-color-hover);
    --onyx-file-upload-small-outline-color: var(--onyx-color-component-focus-neutral);
    --onyx-file-upload-small-dragging-border-color: var(
      --onyx-color-component-border-primary-hover
    );
    --onyx-file-upload-small-dragging-icon-color: var(--onyx-color-text-icons-primary-bold);

    @include input.invalid($formElement: ".onyx-file-upload__input") {
      --onyx-file-upload-border-color: var(--onyx-color-component-border-danger);
      --onyx-file-upload-active-border-color: var(--onyx-color-component-border-danger-hover);
      --onyx-file-upload-error-illustration-display: block;
      --onyx-file-upload-default-illustration-display: none;
      --onyx-file-upload-outline-color: var(--onyx-color-component-focus-danger);
      --onyx-file-upload-active-background-color: var(--onyx-color-base-danger-100);
      --onyx-file-upload-active-label-color: var(--onyx-color-text-icons-danger-intense);
      --onyx-file-upload-hover-svg-background-color-bubble: var(
        --onyx-color-component-border-danger
      );

      --onyx-file-upload-small-hover-background-color: var(--onyx-color-base-background-blank);
      --onyx-file-upload-small-outline-color: var(--onyx-color-component-focus-danger);
      --onyx-file-upload-small-dragging-border-color: var(
        --onyx-color-component-border-danger-hover
      );
      --onyx-file-upload-small-dragging-icon-color: var(--onyx-color-text-icons-danger-bold);
    }

    .onyx-form-element-v2__content {
      height: auto;
    }

    &--large .onyx-form-element-v2__content-skeleton {
      --onyx-form-element-v2-content-height: 7lh;
    }

    &--medium .onyx-form-element-v2__content-skeleton {
      --onyx-form-element-v2-content-height: 2lh;
    }
    &--small .onyx-form-element-v2__content-skeleton {
      --onyx-form-element-v2-content-height: 1lh;
    }

    &.onyx-form-element-v2--label-left,
    &.onyx-form-element-v2--label-right {
      .onyx-form-element-v2__label {
        margin-block: auto;
      }
    }
  }

  .onyx-file-upload-container {
    display: flex;
    flex-direction: column;
    gap: var(--onyx-density-xs);
    position: relative;
    width: 100%;

    & .onyx-tooltip-wrapper {
      width: 100%;
      max-width: 100%;

      & .onyx-tooltip {
        width: 100%;
      }
    }
  }

  .onyx-file-upload {
    all: unset;
    font-family: var(--onyx-font-family-paragraph);
    color: var(--onyx-color-text-icons-neutral-intense);
    border-radius: var(--onyx-radius-md);
    border: var(--onyx-1px-in-rem) dashed var(--onyx-file-upload-border-color);
    background-color: var(--onyx-color-base-background-blank);
    padding: var(--onyx-density-xl);

    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    box-sizing: border-box;
    max-width: 100%;
    width: 100%;
    text-align: center;

    &__icon {
      display: flex;
      gap: var(--onyx-density-2xs);
      color: var(--onyx-color-text-icons-neutral-intense);
      font-family: var(--onyx-font-family-h3);
      font-size: var(--onyx-font-size-md);
      font-weight: var(--onyx-font-weight-semibold);
      line-height: var(--onyx-font-line-height-md);
    }

    &--dragging {
      border-color: var(--onyx-file-upload-active-border-color);
      background-color: var(--onyx-file-upload-active-background-color);

      * {
        // needed to not emit "dragleave" event when hovering over children (e.g. text)
        pointer-events: none;
      }

      .onyx-file-upload__label {
        color: var(--onyx-file-upload-active-label-color);
      }
    }

    .onyx-file-upload-svg {
      margin-bottom: var(--onyx-density-md);
    }

    &__content {
      display: flex;
      flex-direction: column;
      justify-content: center;
      align-items: center;
      gap: var(--onyx-density-2xs);
    }

    &__visually-hidden-input {
      position: absolute;
      width: 100%;
      height: 100%;
      overflow: hidden;
    }

    &__input {
      position: absolute;
      width: 100%;
      height: 100%;
    }

    &.onyx-file-upload--medium {
      padding: var(--onyx-density-xs);
      .onyx-file-upload__icon {
        padding: var(--onyx-density-xs) var(--onyx-density-sm);
      }
    }

    &.onyx-file-upload--small {
      height: calc(
        var(--onyx-form-element-v2-content-height) + 2 * var(--onyx-form-element-v2-padding-block)
      );
      padding: 0 var(--onyx-density-sm);
      border-style: solid;
    }
    &:enabled {
      cursor: pointer;

      &:hover,
      &:focus-within {
        border-color: var(--onyx-file-upload-active-border-color);
        .onyx-file-upload__label {
          color: var(--onyx-file-upload-active-label-color);
        }
      }

      &:hover {
        > .onyx-file-upload-svg {
          --onyx-file-upload-svg-background-color-bubble: var(
            --onyx-file-upload-hover-svg-background-color-bubble
          );
        }
      }

      &:focus-within {
        outline: var(--onyx-outline-width) solid var(--onyx-file-upload-outline-color);
      }

      .onyx-file-upload__text {
        color: var(--onyx-color-text-icons-neutral-medium);
      }

      &.onyx-file-upload--small {
        &:focus-within {
          border: var(--onyx-1px-in-rem) solid var(--onyx-file-upload-border-color);
          outline: var(--onyx-outline-width) solid var(--onyx-file-upload-small-outline-color);
        }

        &.onyx-file-upload--dragging {
          border: var(--onyx-1px-in-rem) dashed var(--onyx-file-upload-small-dragging-border-color);
          .onyx-file-upload__icon {
            color: var(--onyx-file-upload-small-dragging-icon-color);
          }
        }
      }
    }

    &:disabled {
      background-color: var(--onyx-color-base-background-tinted) !important;
      color: var(--onyx-color-text-icons-neutral-soft);
      .onyx-file-upload__icon {
        color: var(--onyx-color-text-icons-neutral-soft);
      }
    }

    &__list {
      --onyx-file-upload-list-gap: var(--onyx-density-2xs);
      display: flex;
      flex-direction: column;
      gap: var(--onyx-file-upload-list-gap);

      &--max-height {
        overflow-y: scroll;

        // the variable values here are given by the OnyxFileCard component
        $file-card-height: OnyxFileCard.height(
          $icon-padding: var(--onyx-density-xs),
          $card-padding: var(--onyx-density-xs),
        );

        max-height: calc(
          (var(--onyx-file-upload-max-files) + 0.5) * $file-card-height +
            (var(--onyx-file-upload-max-files)) * var(--onyx-file-upload-list-gap)
        );
      }
    }

    &__list-button {
      margin-inline: auto;
    }

    &__error-illustration {
      display: var(--onyx-file-upload-error-illustration-display);
    }

    &__default-illustration {
      display: var(--onyx-file-upload-default-illustration-display);
    }
  }
}
</style>
