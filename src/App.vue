<template>
  <div class="macro-editor">
    <h2>坐骑宏生成器</h2>
    <input type="checkbox" value="false" v-model="palMode">
    <label>大领主模式</label>
    <div style="justify-content: space-between; display: flex">
      <div>
        <input type="checkbox" value="false" v-model="buffMode">
        <label>使用以下技能代替下马</label>
      </div>
      <Multiselect class="search-field"
                   v-model="buffName"
                   :options="buffOptions"
                   :disabled="!buffMode"
                   label="name"
                   track-by="name"
                   placeholder="选择技能"
      />
    </div>
    <div v-for="(row, rowIndex) in rows" :key="rowIndex" class="macro-row">
      <h3>行 {{ rowIndex + 1 }}</h3>
      <div v-for="(condition, condIndex) in row.conditions" :key="condIndex" class="macro-condition">
        <label>条件 {{ condIndex + 1 }}:</label>
        <div class="condition-field">
          <label>鼠标: </label>
          <select v-model="condition.btn">
            <option v-for="(label, value) in options.btn" :key="value" :value="value">
              {{ label }}
            </option>
          </select>
        </div>
        <div class="condition-field">
          <label>组合键: </label>
          <select v-model="condition.mod">
            <option v-for="(label, value) in options.mod" :key="value" :value="value">
              {{ label }}
            </option>
          </select>
        </div>
        <div class="condition-field">
          <label>游泳状态: </label>
          <select v-model="condition.swimming">
            <option v-for="(label, value) in options.swimming" :key="value" :value="value">
              {{ label }}
            </option>
          </select>
        </div>
        <div class="condition-field">
          <label>飞行可用: </label>
          <select v-model="condition.flyable">
            <option v-for="(label, value) in options.flyable" :key="value" :value="value">
              {{ label }}
            </option>
          </select>
        </div>
        <div class="condition-field">
          <label>坐骑: </label>
          <Multiselect class="search-field"
                       v-model="condition.mount"
                       :options="mountOptions"
                       :options-limit="5"
                       label="name"
                       track-by="name"
                       placeholder="选择或搜索坐骑"
          />
        </div>
      </div>
      <button @click="addCondition(rowIndex)" class="add-condition-btn">新增条件</button>
    </div>
    <div v-if="showModal" class="modal-overlay">
      <div class="modal-content">
        <h3>生成的宏</h3>
        <textarea readonly>{{ generatedMacro }}</textarea>
        <button @click="copyMacro">复制</button>
        <button @click="closeModal">关闭</button>
      </div>
    </div>
    <button @click="addRow" class="add-row-btn">新增一行</button>
    <button @click="generateMacro" class="generate-macro-btn">生成宏代码</button>
  </div>
</template>

<script>

import Multiselect from "vue-multiselect"
import "vue-multiselect/dist/vue-multiselect.min.css"

export default {
  components: {
    'Multiselect': Multiselect
  },
  data() {
    return {
      showModal: false,
      generatedMacro: '',
      palMode: false,
      buffMode: false,
      buffName: '',
      rows: [
        {conditions: [{btn: 0, mod: 0, swimming: 0, flyable: 0, mount: '',}]}
      ],
      options: {
        btn: {
          0: '（未设定）',
          1: '左键',
          2: '右键'
        },
        mod: {
          0: '（未设定）',
          nomod: '无',
          ctrl: 'Ctrl',
          shift: 'Shift',
          alt: 'Alt'
        },
        swimming: {
          0: '（未设定）',
          1: '正在游泳',
        },
        flyable: {
          0: '（未设定）',
          true: '飞行可用',
        }
      },
      mountOptions: [],
      buffOptions: []
    }
  },
  created() {
    this.loadData()
  },
  methods: {
    async loadData() {
      try {
        const [mountResp, buffResp] = await Promise.all([fetch("/mounts.json"), fetch("/buffs.json")])
        this.buffOptions = await buffResp.json()
        this.mountOptions = await mountResp.json()
      } catch (error) {
        console.error("加载坐骑数据失败:", error)
      }
    },
    addRow() {
      this.rows.push({conditions: []})
    },
    addCondition(rowIndex) {
      this.rows[rowIndex].conditions.push({btn: 0, mod: 0, swimming: 0, flyable: 0, mount: ''})
    },
    generateMacro() {
      const marco = ["#showtooltip",]
      if (this.palMode) {
        marco.push("/cast [noindoors,nomounted,known:十字军光环]!十字军光环;[mounted,known:虔诚光环]!虔诚光环")
      }
      console.log(
          this.buffMode
      )
      console.log(this.buffName.name)
      if (this.buffMode && this.buffName.name?.length) {
        marco.push(`/cast [mounted,known:${this.buffName.name}]${this.buffName.name}`)
      } else {
        marco.push("/dismount [mounted]")
      }
      for (const row of this.rows) {
        const conditions = row.conditions.filter(c => c.mount.name?.length).map(condition => {
          console.log(condition.mount.name?.length)
          const mod = condition.mod === 0 ? "" : `mod:${condition.mod}`
          const swimming = condition.swimming === 0 ? "" : `swimming`
          const flyable = condition.flyable === 0 ? "" : `flyable`
          const btn = condition.btn === 0 ? "" : `button:${condition.btn}`
          const allConditions = [mod, swimming, flyable, btn].filter(v => v !== "").join(",")
          return `${allConditions ? "[" + allConditions + "] " : ""}${condition.mount.name}`
        })
        if (conditions.length > 0) {
          marco.push(`/cast ${conditions.join(";")}`)
        }
      }
      this.generatedMacro = marco.join("\n")// 替换为生成的宏内容
      this.showModal = true
    },
    closeModal() {
      this.showModal = false
    },
    copyMacro() {
      navigator.clipboard.writeText(this.generatedMacro).then(() => {
        alert('宏已复制到剪贴板')
      })
    }
  }
}
</script>

<style scoped>
.macro-editor {
  width: auto;
  min-width: 720px;
}

.macro-row {
  border: 1px solid #ccc;
  padding: 10px;
  margin-bottom: 10px;
}

.macro-condition {
  padding: 5px 0;
}

.condition-field {
  display: flex;
  justify-content: space-between;
  margin-top: 5px;
}

.search-field {
  width: 300px;
}

.add-condition-btn {
  margin-top: 10px;
  cursor: pointer;
}

.add-row-btn {
  margin-top: 20px;
  cursor: pointer;
}

.generate-macro-btn {
  margin-top: 20px;
  margin-left: 20px;
  cursor: pointer;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.modal-content {
  background: white;
  padding: 20px;
  border-radius: 8px;
  width: 300px;
  text-align: center;
}

textarea {
  width: 100%;
  height: 100px;
  margin-top: 10px;
}
</style>
