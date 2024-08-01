<script>
  import { getContext, onDestroy, onMount } from "svelte";
  import Captures from "./Captures.svelte";
  import Loading from "./Loading.svelte";

  export let field;
  export let label;

  const { API, styleable } = getContext("sdk");
  const component = getContext("component");
  const formContext = getContext("form");
  const formStepContext = getContext("form-step");
  const fieldGroupContext = getContext("field-group");

  let loading = false;
  let cameraActive = false;
  let imageCaptured = false;

  let fieldApi;
  let fieldState;

  let blob;
  let imageUrl;
  let cameraStream;
  let videoElement;
  let canvasElement;

  const formApi = formContext?.formApi;
  const labelPos = fieldGroupContext?.labelPosition || "above";
  $: formStep = formStepContext ? $formStepContext || 1 : 1;
  $: formField = formApi?.registerField(
    field,
    "attachment",
    [],
    false,
    null,
    formStep
  );

  $: unsubscribe = formField?.subscribe((value) => {
    fieldState = value?.fieldState;
    fieldApi = value?.fieldApi;
  });

  $: labelClass =
    labelPos === "above" ? "" : `spectrum-FieldLabel--${labelPos}`;

  onMount(() => {
    videoElement = document.getElementById('camera-preview');
    canvasElement = document.getElementById('capture-canvas');
  });

  onDestroy(() => {
    fieldApi?.deregister();
    unsubscribe?.();
    stopCamera();
  });

  const openCamera = async () => {
    loading = true;
    try {
      cameraStream = await navigator.mediaDevices.getUserMedia({
        video: {
          facingMode: { ideal: "environment" },
          width: { ideal: 1920 },
          height: { ideal: 1080 }
        },
      });
      videoElement.srcObject = cameraStream;
      await videoElement.play();
      cameraActive = true;
    } catch (error) {
      console.log(error);
      cameraActive = false;
    }
    loading = false;
  };

  const capture = () => {
    canvasElement.width = videoElement.videoWidth;
    canvasElement.height = videoElement.videoHeight;
    canvasElement.getContext('2d').drawImage(videoElement, 0, 0);
    canvasElement.toBlob((b) => {
      blob = b;
      blob.name = "capture.jpg";
      blob.extension = "jpeg";
      imageUrl = URL.createObjectURL(blob);
      imageCaptured = true;
    }, 'image/jpeg', 0.95);
  };

  const upload = async () => {
    const res = await uploadImage(blob);
    imageUrl = res[0].url;

    const files = fieldState?.value || [];
    files.push(res[0]);
    fieldApi?.setValue(files);
    imageCaptured = false;
    stopCamera();
  };

  const uploadImage = async (image) => {
    let data = new FormData();
    data.append("file", image, "capture.jpg");
    try {
      const res = await API.uploadAttachment({
        data,
        tableId: formContext?.dataSource?.tableId,
      });
      return res;
    } catch (error) {
      return [];
    }
  };

  const retake = () => {
    blob = null;
    imageCaptured = false;
    imageUrl = null;
  };

  const stopCamera = () => {
    if (cameraStream) {
      const tracks = cameraStream.getTracks();
      tracks.forEach(track => track.stop());
    }
    cameraStream = null;
    cameraActive = false;
    imageCaptured = false;
  };
</script>

<div class="spectrum-Form-item" use:styleable={$component.styles}>
  {#if !formContext}
    <div class="placeholder">Form components need to be wrapped in a form</div>
  {:else}
    <label
      class:hidden={!label}
      for={fieldState?.fieldId}
      class={`spectrum-FieldLabel spectrum-FieldLabel--sizeM spectrum-Form-itemLabel ${labelClass}`}
    >
      {label || " "}
    </label>
    <div class="spectrum-Form-itemField">
      {#if !field}
        <div class="error">Please select a field</div>
      {:else}
        <div class="container">
          <Loading {loading} />

          {#if cameraActive}
            <div class="camera-container">
              {#if imageCaptured}
                <img alt="preview" src={imageUrl} />
                <div class="actions">
                  <button
                    on:click={retake}
                    class="spectrum-Button spectrum-Button--fill spectrum-Button--sizeM spectrum-Button--primary"
                    >Retake</button
                  >
                  <button
                    on:click={upload}
                    class="spectrum-Button spectrum-Button--fill spectrum-Button--sizeM spectrum-Button--primary"
                    >Use photo</button
                  >
                </div>
              {:else}
                <video id="camera-preview" autoplay playsinline />
                <canvas id="capture-canvas" style="display: none;"></canvas>
                <div class="capture-button" on:click={capture}>
                  <div class="capture-button-inner"></div>
                </div>
              {/if}
            </div>
          {:else}
            <div class="actions">
              <button
                on:click={openCamera}
                class="spectrum-Button spectrum-Button--fill spectrum-Button--sizeM spectrum-Button--primary"
                >Open camera</button
              >
            </div>
          {/if}

          <Captures attachments={fieldState?.value} />
        </div>

        {#if fieldState?.error}
          <div class="error">{fieldState.error}</div>
        {/if}
      {/if}
    </div>
  {/if}
</div>

<style>
  .camera-container {
    position: relative;
    width: 100%;
    padding-top: 56.25%; /* 16:9 Aspect Ratio */
    overflow: hidden;
  }

  .camera-container video,
  .camera-container img {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .capture-button {
    position: absolute;
    bottom: 20px;
    left: 50%;
    transform: translateX(-50%);
    width: 70px;
    height: 70px;
    border-radius: 50%;
    background-color: rgba(255, 255, 255, 0.3);
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
  }

  .capture-button-inner {
    width: 54px;
    height: 54px;
    border-radius: 50%;
    background-color: white;
  }

  .actions {
    display: flex;
    justify-content: space-between;
    margin-top: 10px;
  }

  .spectrum-ActionButton .spectrum-Icon {
    margin-left: calc(
      -1 * (var(--spectrum-actionbutton-textonly-padding-left-adjusted) -
            var(--spectrum-actionbutton-icononly-padding-left-adjusted))
    ) !important;
  }
</style>