<template>
    <div class="min-h-screen flex items-center justify-center bg-slate-900">
        <div class="max-w-3xl w-full p-6 bg-slate-800 rounded-lg shadow-lg relative">
            <div class="bg-slate-700 p-1.5 rounded-t-lg flex justify-between items-center absolute top-0 left-0 right-0">
                <div class="flex space-x-2 items-center">
                    <button
                        class="rounded-full h-3.5 w-3.5 flex items-center justify-center bg-red-500 text-white hover:bg-red-600"></button>
                    <button
                        class="rounded-full h-3.5 w-3.5 flex items-center justify-center bg-yellow-500 text-white hover:bg-yellow-600"></button>
                    <button
                        class="rounded-full h-3.5 w-3.5 flex items-center justify-center bg-green-500 text-white hover:bg-green-600"></button>
                </div>
            </div>
            <div class="flex space-x-2 mt-10">
                <div v-for="branch in branches" :key="cIndex">
                    <button :class="{ 'bg-slate-800': activeBranch != branch, 'bg-slate-700': activeBranch == branch }"
                        class="rounded-lg px-2 py-1 bg-slate-700 text-stone-50 hover:bg-slate-600 hover:text-stone-100 "
                        @click="setBranch(branch)">{{ branch }}</button>
                </div>
            </div>
            <div class="flex space-x-2 mt-10">
                <div v-for="roadmap in roadmaps" :key="cIndex">
                    <button :class="{ 'bg-slate-800': activeRoadmap != roadmap, 'bg-slate-700': activeRoadmap == roadmap }"
                        class="rounded-lg px-2 py-1 bg-slate-700 text-stone-50 hover:bg-slate-600 hover:text-stone-100 "
                        @click="setRoadmap(roadmap)">{{ roadmap }}</button>
                </div>
            </div>
            <div class="terminal-command mt-10">
                <span class="text-red-300">
                    {{ username }}
                </span>
                <span class="text-yellow-300">
                    @
                </span>
                <span class="text-sky-300">
                    RoadMapApp</span> <span class="text-green-300">
                    <span class="text-orange-300">~ {{ activeBranch }}</span> >> <span class="text-stone-50">cat</span>
                </span>
                <span class="text-teal-500">{{ repo }}/<span class="text-orange-200">{{ activeRoadmap }}</span></span>
                <br>
                <span class="text-stone-50">
                    Displaying roadmap....
                </span>
            </div>
            <p v-if="$fetchState.pending">Fetching map data...</p>
            <p v-else-if="$fetchState.error">An error occurred :(</p>
            <div v-else>
                <!-- Display list of sections -->
                <div v-for="(entry, index) in sections" :key="index" class="mb-6 mt-10 text-stone-50">
                    <h1 class="text-3xl font-bold text-cyan-green-neon mb-2">{{ entry.title }}</h1>
                    <div v-if="entry.content.length > 0">
                        <ul class="list-disc pl-6">
                            <li v-for="(contentItem, cIndex) in entry.content" :key="cIndex">
                                <div v-html="$md.render(contentItem)"></div>
                            </li>
                        </ul>
                    </div>
                    <div v-for="(section, sIndex) in entry.sections" :key="sIndex" class="ml-4">
                        <h3 class="text-lg font-semibold mb-1">{{ section.title }}</h3>
                        <div v-if="section.content.length > 0">
                            <ul class="list-disc pl-6">
                                <li v-for="(contentItem, cIndex) in section.content" :key="cIndex">
                                    <div v-html="$md.render(contentItem)"></div>
                                </li>
                            </ul>
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>



<script>
export default {

    data() {
        return {
            username: "",
            repo: "",
            mdData: "",
            activeBranch: "",
            activeRoadmap: "",
            branches: [],
            roadmaps: [],
            sections: [],

        };
    }, computed: {
        console: () => console,
        window: () => window,
    },
    methods: {
        async setBranch(branch) {
            this.console.log(branch);
            this.activeBranch = branch;
            await this.getRoadmaps();
            await this.getMakdown();

        },
        async setRoadmap(roadmap) {
            this.console.log(roadmap);
            this.activeRoadmap = roadmap;
            await this.getMakdown();
        },

        async getBranches() {
            // https://api.github.com/repos/denver-code/roadmaptest/branches
            this.branches = [];
            const link = `https://api.github.com/repos/${this.username}/${this.repo}/branches`;
            const response = await fetch(link);
            var _branches = await response.json();
            _branches.forEach(branch => {
                this.branches.push(branch.name);
            });
            this.activeBranch = this.branches[0];

        },
        async getRoadmaps() {
            // https://api.github.com/repos/denver-code/roadmaptest/contents
            this.roadmaps = [];
            const link = `https://api.github.com/repos/${this.username}/${this.repo}/contents?ref=${this.activeBranch}`;
            const response = await fetch(link);
            var _roadmaps = await response.json();
            _roadmaps.forEach(roadmap => {
                if (roadmap.name.toLowerCase() == "roadmap.md" || roadmap.name.toLowerCase() == "changelog.md") {
                    this.roadmaps.push(roadmap.name);
                }
            });
            this.activeRoadmap = this.roadmaps[0];
        },
        async getMakdown() {
            // https://raw.githubusercontent.com/denver-code/roadmaptest/main/roadmap.md
            const link = `https://raw.githubusercontent.com/${this.username}/${this.repo}/${this.activeBranch}/${this.activeRoadmap}`;
            const response = await fetch(link);
            this.mdData = await response.text();
            const lines = this.mdData.split("\n");
            let changelog = [];
            let currentEntry = null;

            for (const line of lines) {
                if (line.match(/^##\s+.+/)) {
                    if (currentEntry) changelog.push(currentEntry);
                    const entryTitle = line.replace("##", "").trim();
                    currentEntry = {
                        title: entryTitle,
                        content: [],
                        sections: [],
                    };
                } else if (line.match(/^###\s+.+/)) {
                    const sectionTitle = line.replace("###", "").trim();
                    currentEntry.sections.push({
                        title: sectionTitle,
                        content: [],
                    });
                } else if (currentEntry && currentEntry.sections.length > 0 && line.trim() !== "") {
                    const lastIndex = currentEntry.sections.length - 1;
                    currentEntry.sections[lastIndex].content.push(line.trim());
                } else if (currentEntry && line.trim() !== "") {
                    currentEntry.content.push(line.trim());
                }
            }

            if (currentEntry) changelog.push(currentEntry);

            this.sections = changelog;

        }
    },
    async fetch() {
        this.username = this.$route.params.user;
        this.repo = this.$route.params.repo;
        await this.getBranches();
        await this.getRoadmaps();
        await this.getMakdown();
    },

};
</script>
  

<style scoped>
.text-cyan-green-neon {
    color: #00ffcc;
    text-shadow: 0 0 5px rgba(0, 255, 204, 0.8);
}

.terminal-command {
    font-family: 'Courier New', monospace;
    background-color: #000;
    padding: 10px;
    border-radius: 5px;
    box-shadow: 0 3px 6px rgba(0, 0, 0, 0.1);
}
</style>

