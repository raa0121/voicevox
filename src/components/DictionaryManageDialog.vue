<template>
  <q-dialog
    maximized
    seamless
    transition-show="jump-up"
    transition-hide="jump-down"
    class="setting-dialog"
    v-model="dictionaryManageDialogOpenedComputed"
  >
    <q-layout container view="hHh Lpr fFf" class="bg-background">
      <q-page-container>
        <q-header class="q-pa-sm">
          <q-toolbar>
            <q-toolbar-title class="text-display"
              >読み方＆アクセント辞書</q-toolbar-title
            >
            <q-space />
            <!-- close button -->
            <q-btn
              round
              flat
              icon="close"
              color="display"
              @click="discardOrNotDialog(closeDialog)"
            />
          </q-toolbar>
        </q-header>
        <q-page class="row">
          <div v-if="loadingDict" class="loading-dict">
            <div>
              <q-spinner color="primary" size="2.5rem" />
              <div class="q-mt-xs">読み込み中・・・</div>
            </div>
          </div>
          <div class="col-4 word-list-col">
            <div v-if="wordEditing" class="word-list-disable-overlay" />
            <div class="word-list-header">
              <div class="word-list-title text-h5">単語一覧</div>
              <div>
                <q-btn
                  outline
                  text-color="warning"
                  class="text-no-wrap text-bold"
                  @click="deleteWord"
                  :disable="uiLocked || !isDeletable"
                  >削除</q-btn
                >
              </div>
              <div>
                <q-btn
                  outline
                  text-color="display"
                  class="text-no-wrap text-bold"
                  @click="editWord"
                  :disable="uiLocked || !selectedId"
                  >編集</q-btn
                >
              </div>
              <div>
                <q-btn
                  outline
                  text-color="display"
                  class="text-no-wrap text-bold"
                  @click="newWord"
                  :disable="uiLocked"
                  >追加</q-btn
                >
              </div>
            </div>
            <q-list class="word-list">
              <q-item
                v-for="(value, key) in userDict"
                :key="key"
                tag="label"
                v-ripple
                clickable
                @click="selectWord(key)"
                :active="selectedId === key"
                active-class="active-word"
              >
                <q-item-section>
                  <q-item-label class="text-display">{{
                    value.surface
                  }}</q-item-label>
                  <q-item-label caption>{{ value.yomi }}</q-item-label>
                </q-item-section>
              </q-item>
            </q-list>
          </div>

          <!-- 右側のpane -->
          <div v-if="wordEditing" class="col-8 no-wrap text-no-wrap">
            <div class="row q-pl-md q-mt-md">
              <div class="text-h6">単語</div>
              <q-input
                ref="surfaceInput"
                class="word-input"
                v-model="surface"
                @blur="setSurface(surface)"
                @keydown="yomiFocusWhenEnter"
                dense
                :disable="uiLocked"
              />
            </div>
            <div class="row q-pl-md q-pt-sm">
              <div class="text-h6">読み</div>
              <q-input
                ref="yomiInput"
                class="word-input"
                v-model="yomi"
                @blur="setYomi(yomi)"
                @keydown="setYomiWhenEnter"
                dense
                :error="!isOnlyHiraOrKana"
                :disable="uiLocked"
              >
                <template v-slot:error>
                  読みに使える文字はひらがなとカタカナのみです。
                </template>
              </q-input>
            </div>
            <div class="row q-pl-md q-pt-sm text-h6">アクセント調整</div>
            <div class="row q-pl-md accent-desc">
              語尾のアクセントを考慮するため、「が」が自動で挿入されます。
            </div>
            <div class="row q-px-md" style="height: 130px">
              <div class="play-button">
                <q-btn
                  v-if="!nowPlaying && !nowGenerating"
                  fab
                  color="primary-light"
                  text-color="display-dark"
                  icon="play_arrow"
                  @click="play"
                />
                <q-btn
                  v-else
                  fab
                  color="primary-light"
                  text-color="display-dark"
                  icon="stop"
                  @click="stop"
                  :disable="nowGenerating"
                />
              </div>
              <div
                ref="accentPhraseTable"
                class="accent-phrase-table overflow-hidden-y"
                :style="{
                  justifyContent: centeringAccentPhrase ? 'center' : undefined,
                }"
              >
                <div v-if="accentPhrase" class="mora-table">
                  <audio-accent
                    :accent-phrase="accentPhrase"
                    :accent-phrase-index="0"
                    :ui-locked="uiLocked"
                    @changeAccent="changeAccent"
                  />
                  <template
                    v-for="(mora, moraIndex) in accentPhrase.moras"
                    :key="moraIndex"
                  >
                    <div
                      class="text-cell"
                      :style="{
                        gridColumn: `${moraIndex * 2 + 1} / span 1`,
                      }"
                    >
                      {{ mora.text }}
                    </div>
                    <div
                      v-if="moraIndex < accentPhrase.moras.length - 1"
                      class="splitter-cell"
                      :style="{
                        gridColumn: `${moraIndex * 2 + 2} / span 1`,
                      }"
                    />
                  </template>
                </div>
              </div>
            </div>
            <div class="row q-px-md save-delete-reset-buttons">
              <q-space />
              <q-btn
                v-show="!!selectedId"
                outline
                text-color="display"
                class="text-no-wrap text-bold q-mr-sm"
                @click="resetWord"
                :disable="uiLocked || !isWordChanged"
                >リセット</q-btn
              >
              <q-btn
                outline
                text-color="display"
                class="text-no-wrap text-bold q-mr-sm"
                @click="saveWord"
                :disable="uiLocked || !isWordChanged"
                >保存</q-btn
              >
              <q-btn
                outline
                text-color="display"
                class="text-no-wrap text-bold q-mr-sm"
                @click="isWordChanged ? discardOrNotDialog(cancel) : cancel()"
                :disable="uiLocked"
                >キャンセル</q-btn
              >
            </div>
          </div>
        </q-page>
      </q-page-container>
    </q-layout>
  </q-dialog>
</template>

<script lang="ts">
import { computed, defineComponent, nextTick, ref, watch } from "vue";
import { useStore } from "@/store";
import { AccentPhrase, AudioQuery, UserDictWord } from "@/openapi";
import {
  convertHiraToKana,
  convertLongVowel,
  createKanaRegex,
} from "@/store/utility";
import AudioAccent from "@/components/AudioAccent.vue";
import { QInput, useQuasar } from "quasar";
import { AudioItem } from "@/store/type";

export default defineComponent({
  name: "DictionaryManageDialog",
  components: { AudioAccent },
  props: {
    modelValue: {
      type: Boolean,
      required: true,
    },
  },

  setup(props, { emit }) {
    const store = useStore();
    const $q = useQuasar();

    const dictionaryManageDialogOpenedComputed = computed({
      get: () => props.modelValue,
      set: (val) => emit("update:modelValue", val),
    });
    const uiLocked = computed(() => store.getters.UI_LOCKED);
    const nowGenerating = ref(false);
    const nowPlaying = ref(false);

    const loadingDict = ref(false);
    const userDict = ref<Record<string, UserDictWord>>({});

    const loadingDictProcess = async () => {
      loadingDict.value = true;
      try {
        userDict.value = await store.dispatch("LOAD_USER_DICT");
      } catch {
        $q.dialog({
          title: "辞書の取得に失敗しました",
          message: "エンジンの再起動をお試しください。",
          ok: {
            label: "閉じる",
            flat: true,
            textColor: "display",
          },
        }).onOk(() => {
          dictionaryManageDialogOpenedComputed.value = false;
        });
      }
      loadingDict.value = false;
    };
    watch(dictionaryManageDialogOpenedComputed, async (newValue) => {
      if (newValue) {
        await loadingDictProcess();
        toInitialState();
      }
    });

    const wordEditing = ref(false);

    const surfaceInput = ref<QInput>();
    const yomiInput = ref<QInput>();
    const yomiFocusWhenEnter = (event: KeyboardEvent) => {
      // keyCodeは非推奨で、keyが推奨だが、
      // key === "Enter"はIMEのEnterも拾ってしまうので、keyCodeを用いている
      if (event.keyCode === 13) {
        yomiInput.value?.focus();
      }
    };
    const setYomiWhenEnter = (event: KeyboardEvent) => {
      // keyCodeは非推奨で、keyが推奨だが、
      // key === "Enter"はIMEのEnterも拾ってしまうので、keyCodeを用いている
      if (event.keyCode === 13) {
        setYomi(yomi.value);
      }
    };

    const selectedId = ref("");
    const surface = ref("");
    const yomi = ref("");

    const styleId = computed(() => {
      if (!store.getters.USER_ORDERED_CHARACTER_INFOS) return 0;
      return store.getters.USER_ORDERED_CHARACTER_INFOS[0].metas.styles[0]
        .styleId;
    });

    const kanaRegex = createKanaRegex();
    const isOnlyHiraOrKana = ref(true);
    const accentPhrase = ref<AccentPhrase | undefined>();
    const accentPhraseTable = ref<HTMLElement>();
    // スクロールしなければならないほど長いアクセント句はセンタリングしないようにする
    // computedにすると、ダイアログを表示した際にしか動作しないので、refにして変更時に代入する
    const centeringAccentPhrase = ref(true);
    // FIXME: CSSで完結させる
    const computeCenteringAccentPhrase = () => {
      centeringAccentPhrase.value =
        !!accentPhraseTable.value &&
        accentPhraseTable.value.scrollWidth ==
          accentPhraseTable.value.offsetWidth;
    };
    const convertHankakuToZenkaku = (text: string) => {
      // "!"から"~"までの範囲の文字(数字やアルファベット)を全角に置き換える
      return text.replace(/[\u0021-\u007e]/g, (s) => {
        return String.fromCharCode(s.charCodeAt(0) + 0xfee0);
      });
    };
    const setSurface = (text: string) => {
      // surfaceを全角化する
      // 入力は半角でも問題ないが、登録時に全角に変換され、isWordChangedの判断がおかしくなることがあるので、
      // 入力後に自動で変換するようにする
      surface.value = convertHankakuToZenkaku(text);
    };
    const setYomi = async (text: string, changeWord?: boolean) => {
      // テキスト長が0の時にエラー表示にならないように、テキスト長を考慮する
      isOnlyHiraOrKana.value = !text.length || kanaRegex.test(text);
      // 読みが変更されていない場合は、アクセントフレーズに変更を加えない
      // ただし、読みが同じで違う単語が存在する場合が考えられるので、changeWordフラグを考慮する
      // 「ガ」が自動挿入されるので、それを考慮してsliceしている
      if (
        text ==
          accentPhrase.value?.moras
            .map((v) => v.text)
            .join("")
            .slice(0, -1) &&
        !changeWord
      ) {
        return;
      }
      if (isOnlyHiraOrKana.value && text.length) {
        text = convertHiraToKana(text);
        text = convertLongVowel(text);
        accentPhrase.value = (
          await store.dispatch("FETCH_ACCENT_PHRASES", {
            text: text + "ガ'",
            styleId: styleId.value,
            isKana: true,
          })
        )[0];
        if (
          selectedId.value &&
          userDict.value[selectedId.value].yomi === text
        ) {
          accentPhrase.value.accent = computeDisplayAccent();
        }
      } else {
        accentPhrase.value = undefined;
      }
      yomi.value = text;
      await nextTick();
      computeCenteringAccentPhrase();
    };
    window.onresize = computeCenteringAccentPhrase;

    const changeAccent = async (_: number, accent: number) => {
      if (accentPhrase.value) {
        accentPhrase.value.accent = accent;
        accentPhrase.value = (
          await store.dispatch("FETCH_MORA_DATA", {
            accentPhrases: [accentPhrase.value],
            styleId: styleId.value,
          })
        )[0];
      }
    };

    const audioElem = new Audio();
    audioElem.pause();

    const play = async () => {
      if (!accentPhrase.value) return;
      nowGenerating.value = true;
      const query: AudioQuery = {
        accentPhrases: [accentPhrase.value],
        speedScale: 1.0,
        pitchScale: 0,
        intonationScale: 1.0,
        volumeScale: 1.0,
        prePhonemeLength: 0.1,
        postPhonemeLength: 0.1,
        outputSamplingRate: store.state.savingSetting.outputSamplingRate,
        outputStereo: store.state.savingSetting.outputStereo,
      };

      const audioItem: AudioItem = {
        text: yomi.value,
        styleId: styleId.value,
        query,
      };

      let blob = await store.dispatch("GET_AUDIO_CACHE_FROM_AUDIO_ITEM", {
        audioItem,
      });
      if (!blob) {
        blob = await store.dispatch("GENERATE_AUDIO_FROM_AUDIO_ITEM", {
          audioItem,
        });
        if (!blob) {
          nowGenerating.value = false;
          $q.dialog({
            title: "生成に失敗しました",
            message: "エンジンの再起動をお試しください。",
            ok: {
              label: "閉じる",
              flat: true,
              textColor: "display",
            },
          });
          return;
        }
      }
      nowGenerating.value = false;
      nowPlaying.value = true;
      await store.dispatch("PLAY_AUDIO_BLOB", { audioElem, audioBlob: blob });
      nowPlaying.value = false;
    };
    const stop = () => {
      audioElem.pause();
    };

    // accent phraseにあるaccentと実際に登録するアクセントには差が生まれる
    // アクセントが自動追加される「ガ」に指定されている場合、
    // 実際に登録するaccentの値は0となるので、そうなるように処理する
    const computeRegisteredAccent = () => {
      if (!accentPhrase.value) throw new Error();
      let accent = accentPhrase.value.accent;
      accent = accent === accentPhrase.value.moras.length ? 0 : accent;
      return accent;
    };
    // computeの逆
    // 辞書から得たaccentが0の場合に、自動で追加される「ガ」の位置にアクセントを表示させるように処理する
    const computeDisplayAccent = () => {
      if (!accentPhrase.value || !selectedId.value) throw new Error();
      let accent = userDict.value[selectedId.value].accentType;
      accent = accent === 0 ? accentPhrase.value.moras.length : accent;
      return accent;
    };

    // 操作（ステートの移動）
    const isWordChanged = computed(() => {
      if (selectedId.value === "") {
        return surface.value && yomi.value && accentPhrase.value;
      }
      // 一旦代入することで、userDictそのものが更新された時もcomputedするようにする
      const dict = userDict.value;
      const dictData = dict[selectedId.value];
      return (
        dictData &&
        (dictData.surface !== surface.value ||
          dictData.yomi !== yomi.value ||
          dictData.accentType !== computeRegisteredAccent())
      );
    });
    const saveWord = async () => {
      if (!accentPhrase.value) throw new Error(`accentPhrase === undefined`);
      const accent = computeRegisteredAccent();
      if (selectedId.value) {
        try {
          await store.dispatch("REWRITE_WORD", {
            wordUuid: selectedId.value,
            surface: surface.value,
            pronunciation: yomi.value,
            accentType: accent,
          });
        } catch {
          $q.dialog({
            title: "単語の更新に失敗しました",
            message: "エンジンの再起動をお試しください。",
            ok: {
              label: "閉じる",
              flat: true,
              textColor: "display",
            },
          });
          return;
        }
      } else {
        try {
          await store.dispatch("ADD_WORD", {
            surface: surface.value,
            pronunciation: yomi.value,
            accentType: accent,
          });
        } catch {
          $q.dialog({
            title: "単語の登録に失敗しました",
            message: "エンジンの再起動をお試しください。",
            ok: {
              label: "閉じる",
              flat: true,
              textColor: "display",
            },
          });
          return;
        }
      }
      await loadingDictProcess();
      toInitialState();
    };
    const isDeletable = computed(() => !!selectedId.value);
    const deleteWord = () => {
      $q.dialog({
        title: "登録された単語を削除しますか？",
        message: "削除された単語は復旧できません。",
        persistent: true,
        focus: "cancel",
        ok: {
          label: "削除",
          flat: true,
          textColor: "warning",
        },
        cancel: {
          label: "キャンセル",
          flat: true,
          textColor: "display",
        },
      }).onOk(async () => {
        try {
          await store.dispatch("DELETE_WORD", {
            wordUuid: selectedId.value,
          });
        } catch {
          $q.dialog({
            title: "単語の削除に失敗しました",
            message: "エンジンの再起動をお試しください。",
            ok: {
              label: "閉じる",
              flat: true,
              textColor: "display",
            },
          });
          return;
        }
        await loadingDictProcess();
        toInitialState();
      });
    };
    const resetWord = () => {
      $q.dialog({
        title: "単語の変更をリセットしますか？",
        message: "単語の変更は破棄されてリセットされます。",
        persistent: true,
        focus: "cancel",
        ok: {
          label: "リセット",
          flat: true,
          textColor: "display",
        },
        cancel: {
          label: "キャンセル",
          flat: true,
          textColor: "display",
        },
      }).onOk(() => {
        toWordEditingState();
      });
    };
    const discardOrNotDialog = (okCallback: () => void) => {
      if (isWordChanged.value) {
        $q.dialog({
          title: "単語の追加・変更を破棄しますか？",
          message:
            "このまま続行すると、単語の追加・変更は破棄されてリセットされます。",
          persistent: true,
          focus: "cancel",
          ok: {
            label: "続行",
            flat: true,
            textColor: "display",
          },
          cancel: {
            label: "キャンセル",
            flat: true,
            textColor: "display",
          },
        }).onOk(okCallback);
      } else {
        okCallback();
      }
    };
    const newWord = () => {
      selectedId.value = "";
      surface.value = "";
      setYomi("");
      editWord();
    };
    const editWord = () => {
      toWordEditingState();
    };
    const selectWord = (id: string) => {
      selectedId.value = id;
      surface.value = userDict.value[id].surface;
      setYomi(userDict.value[id].yomi, true);
      toWordSelectedState();
    };
    const cancel = () => {
      toInitialState();
    };
    const closeDialog = () => {
      toDialogClosedState();
    };

    // ステートの移動
    // 初期状態
    const toInitialState = () => {
      wordEditing.value = false;
      selectedId.value = "";
      surface.value = "";
      setYomi("");
    };
    // 単語が選択されているだけの状態
    const toWordSelectedState = () => {
      wordEditing.value = false;
    };
    // 単語が編集されている状態
    const toWordEditingState = () => {
      wordEditing.value = true;
      surfaceInput.value?.focus();
    };
    // ダイアログが閉じている状態
    const toDialogClosedState = () => {
      dictionaryManageDialogOpenedComputed.value = false;
    };

    return {
      dictionaryManageDialogOpenedComputed,
      uiLocked,
      nowGenerating,
      nowPlaying,
      userDict,
      loadingDict,
      wordEditing,
      surfaceInput,
      yomiInput,
      yomiFocusWhenEnter,
      setYomiWhenEnter,
      selectedId,
      surface,
      yomi,
      selectWord,
      newWord,
      editWord,
      cancel,
      isOnlyHiraOrKana,
      setSurface,
      setYomi,
      accentPhrase,
      accentPhraseTable,
      centeringAccentPhrase,
      changeAccent,
      play,
      stop,
      isWordChanged,
      isDeletable,
      saveWord,
      deleteWord,
      resetWord,
      closeDialog,
      discardOrNotDialog,
    };
  },
});
</script>

<style lang="scss" scoped>
@use '@/styles/colors' as colors;
@use '@/styles/variables' as vars;

.word-list-col {
  border-right: solid 1px colors.$setting-item;
  position: relative; // オーバーレイのため
}

.word-list-header {
  margin: 1rem;

  display: flex;
  gap: 0.5rem;
  align-items: center;
  justify-content: space-between;
  .word-list-title {
    flex-grow: 1;
  }
}

.word-list {
  // menubar-height + header-height + window-border-width +
  // 46(title) + 52(new word button)
  height: calc(
    100vh - #{vars.$menubar-height + vars.$header-height +
      vars.$window-border-width + 46px + 52px}
  );
  width: 100%;
  overflow-y: auto;
}

.active-word {
  background: rgba(colors.$primary-rgb, 0.4);
}

.loading-dict {
  position: absolute;
  inset: 0;
  z-index: 10;
  display: flex;
  text-align: center;
  align-items: center;
  justify-content: center;

  > div {
    color: colors.$display-dark;
    background: colors.$background-light;
    border-radius: 6px;
    padding: 14px;
  }
}

.word-list-disable-overlay {
  background-color: rgba($color: #000000, $alpha: 0.4);
  width: 100%;
  height: 100%;
  position: absolute;
  z-index: 10;
}

.word-input {
  padding-left: 10px;
  width: calc(66vw - 80px);

  :deep(.q-field__control) {
    height: 2rem;
  }

  :deep(.q-placeholder) {
    padding: 0;
    font-size: 20px;
  }

  :deep(.q-field__after) {
    height: 2rem;
  }
}

.accent-desc {
  color: rgba(colors.$display-rgb, 0.5);
  font-size: 12px;
}

.play-button {
  margin: auto 0;
  padding-right: 16px;
}

.accent-phrase-table {
  flex-grow: 1;
  align-self: stretch;

  display: flex;
  overflow-x: scroll;
  width: calc(66vw - 140px);

  .mora-table {
    display: inline-grid;
    align-self: stretch;
    grid-template-rows: 1fr 60px 30px;

    .text-cell {
      padding: 0;
      min-width: 30px;
      max-width: 30px;
      grid-row-start: 3;
      text-align: center;
      color: colors.$display;
    }

    .splitter-cell {
      min-width: 10px;
      max-width: 10px;
      grid-row: 3 / span 1;
      z-index: vars.$detail-view-splitter-cell-z-index;
    }
  }
}

.save-delete-reset-buttons {
  // menubar-height + header-height + window-border-width
  // 46(surface input) + 58(yomi input) + 38(accent title) + 18(accent desc) + 130(accent)
  height: calc(
    100vh - #{vars.$menubar-height + vars.$header-height +
      vars.$window-border-width + 46px + 58px + 38px + 18px + 130px}
  );
  padding: 20px;

  display: flex;
  align-items: flex-end;
}
</style>
