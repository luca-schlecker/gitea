<script lang="ts" setup>
import {computed, nextTick, onMounted, shallowRef, useTemplateRef, type ShallowRef} from 'vue';
import SvgIcon from './SvgIcon.vue';
import {GET} from '../modules/fetch.ts';
import {urlQueryEscape} from '../utils/url.ts';
import type {SvgName} from '../svg.ts';

const {appSubUrl, pageData} = window.config;

type DashboardRepo = {
  id: number,
  link: string,
  full_name: string,
  archived: boolean,
  fork: boolean,
  mirror: boolean,
  template: boolean,
  private: boolean,
  internal: boolean,
  latest_commit_status_state?: CommitStatus,
  latest_commit_status_state_link?: string,
  locale_latest_commit_status_state?: string,
};

type FolderRepoItem = {
  kind: "repo",
  name: string,
  repo: DashboardRepo,
};
type FolderSubfolderItem = {
  kind: "folder",
  name: string,
  children: FolderItem[],
};
type FolderItem = FolderRepoItem | FolderSubfolderItem;

type CommitStatus = 'pending' | 'success' | 'error' | 'failure' | 'warning' | 'skipped';

type WebSearchRepo = {
  repository: DashboardRepo,
  latest_commit_status: {
    State: CommitStatus,
    TargetURL: string,
  } | null,
  locale_latest_commit_status: string,
};

type CommitStatusMap = {
  [status in CommitStatus]: {
    name: SvgName,
    color: string,
  };
};

type Tab = 'repos' | 'organizations' | 'folders';
type RepoFilter = 'all' | 'forks' | 'mirrors' | 'sources' | 'collaborative';
type ArchivedFilter = 'archived' | 'unarchived' | 'both';
type PrivateFilter = 'private' | 'public' | 'both';

// make sure this matches templates/repo/commit_status.tmpl
const commitStatus: CommitStatusMap = {
  pending: {name: 'octicon-dot-fill', color: 'tw-text-yellow'},
  success: {name: 'octicon-check', color: 'tw-text-green'},
  error: {name: 'gitea-exclamation', color: 'tw-text-red'},
  failure: {name: 'octicon-x', color: 'tw-text-red'},
  warning: {name: 'gitea-exclamation', color: 'tw-text-yellow'},
  skipped: {name: 'octicon-skip', color: 'tw-text-text-light'},
};

const searchModes = new Map<RepoFilter, string>([
  ['all', ''],
  ['forks', 'fork'],
  ['mirrors', 'mirror'],
  ['sources', 'source'],
  ['collaborative', 'collaborative'],
]);

const pageDataDefaults = {
  subUrl: appSubUrl,
  organizations: [] as Array<{name: string, full_name: string, num_repos: number, org_visibility: string}>,
  avatars: {} as Record<string, string>,
  isOrganization: true,
  canCreateOrganization: false,
  organizationsTotalCount: 0,
  organizationId: 0,
  searchLimit: 0,
  uid: 0,
  teamId: 0,
  isMirrorsEnabled: false,
  textNoOrg: '',
  textNoRepo: '',
  textRepository: '',
  textOrganization: '',
  textMyRepos: '',
  textNewRepo: '',
  textSearchRepos: '',
  textFilter: '',
  textShowArchived: '',
  textShowPrivate: '',
  textShowBothArchivedUnarchived: '',
  textShowOnlyUnarchived: '',
  textShowOnlyArchived: '',
  textShowBothPrivatePublic: '',
  textShowOnlyPublic: '',
  textShowOnlyPrivate: '',
  textAll: '',
  textSources: '',
  textForks: '',
  textMirrors: '',
  textCollaborative: '',
  textFirstPage: '',
  textPreviousPage: '',
  textNextPage: '',
  textLastPage: '',
  textMyOrgs: '',
  textNewOrg: '',
  textOrgVisibilityLimited: '',
  textOrgVisibilityPrivate: '',
};

const {
  subUrl, organizations, avatars, isOrganization, canCreateOrganization, organizationsTotalCount, organizationId,
  searchLimit, uid, teamId, isMirrorsEnabled,
  textNoOrg, textNoRepo, textRepository, textOrganization, textMyRepos, textNewRepo, textSearchRepos,
  textFilter, textShowArchived, textShowPrivate,
  textShowBothArchivedUnarchived, textShowOnlyUnarchived, textShowOnlyArchived,
  textShowBothPrivatePublic, textShowOnlyPublic, textShowOnlyPrivate,
  textAll, textSources, textForks, textMirrors, textCollaborative,
  textFirstPage, textPreviousPage, textNextPage, textLastPage,
  textMyOrgs, textNewOrg, textOrgVisibilityLimited, textOrgVisibilityPrivate,
}: typeof pageDataDefaults = {...pageDataDefaults, ...pageData.dashboardRepoList};

const textArchivedFilterTitles = new Map<ArchivedFilter, string>([
  ['archived', textShowOnlyArchived],
  ['unarchived', textShowOnlyUnarchived],
  ['both', textShowBothArchivedUnarchived],
]);

const textPrivateFilterTitles = new Map<PrivateFilter, string>([
  ['private', textShowOnlyPrivate],
  ['public', textShowOnlyPublic],
  ['both', textShowBothPrivatePublic],
]);

const initialParams = new URLSearchParams(window.location.search);
const tab = shallowRef((initialParams.get('repo-search-tab') || 'repos') as Tab);
const reposFilter = shallowRef((initialParams.get('repo-search-filter') || 'all') as RepoFilter);
const privateFilter = shallowRef((initialParams.get('repo-search-private') || 'both') as PrivateFilter);
const archivedFilter = shallowRef((initialParams.get('repo-search-archived') || 'unarchived') as ArchivedFilter);
const searchQuery = shallowRef(initialParams.get('repo-search-query') || '');
const page = shallowRef(Number(initialParams.get('repo-search-page')) || 1);

const repos = shallowRef<DashboardRepo[]>([]);
const reposTotalCount = shallowRef<number | null>(null);
const finalPage = shallowRef(1);
const counts = shallowRef<Record<string, number>>({});
const isLoading = shallowRef(false);
const initialSearchDone = shallowRef(false);
const activeIndex = shallowRef(-1); // don't select anything at load, first cursor down will select

const elSearch = useTemplateRef('elSearch') as Readonly<ShallowRef<HTMLInputElement>>;

const countsKey = computed(() => `${reposFilter.value}:${archivedFilter.value}:${privateFilter.value}`);
const showMoreReposLink = computed(() => repos.value.length > 0 && repos.value.length < repoTypeCount.value);
const repoTypeCount = computed(() => counts.value[countsKey.value]);
const checkboxArchivedFilterTitle = computed(() => textArchivedFilterTitles.get(archivedFilter.value));
const checkboxArchivedFilterProps = computed(() => ({checked: archivedFilter.value === 'archived', indeterminate: archivedFilter.value === 'both'}));
const checkboxPrivateFilterTitle = computed(() => textPrivateFilterTitles.get(privateFilter.value));
const checkboxPrivateFilterProps = computed(() => ({checked: privateFilter.value === 'private', indeterminate: privateFilter.value === 'both'}));

// unknown query string values fall back to no mode
const searchMode = computed(() => searchModes.get(reposFilter.value) ?? '');

const searchURL = computed(() => {
  // unknown query string values send no filter
  const archived = archivedFilter.value === 'archived' ? '&archived=true' : archivedFilter.value === 'unarchived' ? '&archived=false' : '';
  const isPrivate = privateFilter.value === 'private' ? '&is_private=true' : privateFilter.value === 'public' ? '&is_private=false' : '';
  return `${subUrl}/repo/search?sort=updated&order=desc&uid=${uid}&team_id=${teamId}&q=${urlQueryEscape(searchQuery.value)}` +
    `&page=${page.value}&limit=${searchLimit}&mode=${searchMode.value}${archived}${isPrivate}`;
});

const folderSeparator: RegExp = /\/|\./;
const currentFolderPath = shallowRef<string[]>(JSON.parse(initialParams.get('repo-folder') || '[]'));
const fullFolderTree = shallowRef<FolderItem[]>([]);
const currentFolder = computed(() => {
  let currentSubtree = fullFolderTree.value as FolderItem[];

  for (const part of currentFolderPath.value as string[]) {
    const nextSubtree = currentSubtree.filter(it => it.kind === "folder").find(it => it.name === part);
    if (nextSubtree === undefined) { break; }
    else { currentSubtree = nextSubtree.children; }
  }

  return currentSubtree;
});
const getAllReposURL = computed(() => {
  const archived = archivedFilter.value === 'archived' ? '&archived=true' : archivedFilter.value === 'unarchived' ? '&archived=false' : '';
  const isPrivate = privateFilter.value === 'private' ? '&is_private=true' : privateFilter.value === 'public' ? '&is_private=false' : '';
  return `${subUrl}/repo/search?uid=${uid}&team_id=${teamId}&q=${urlQueryEscape(searchQuery.value)}&limit=18446744073709551615&mode=${searchMode.value}${archived}${isPrivate}`;
});

async function loadFolderTree() {
  const request = GET(getAllReposURL.value);
  const response = await request;
  const json: { data: WebSearchRepo[] } = await response.json();

  let tree: FolderItem[] = [];

  for (const repo of json.data) {
    let currentSubtree = tree;
    const parts = repo.repository.full_name.split(folderSeparator);

    for (let i = 0; i < parts.length - 1; i++) {
      const nextSubtree = currentSubtree.filter(it => it.kind === "folder").find(it => it.name === parts[i]);
      if (nextSubtree === undefined) {
        const newFolder: FolderSubfolderItem = {
          kind: "folder",
          name: parts[i],
          children: [],
        };
        currentSubtree.push(newFolder);
        currentSubtree = newFolder.children;
      } else {
        currentSubtree = nextSubtree.children;
      }
    }

    currentSubtree.push({
      kind: "repo",
      name: parts.at(-1) || "",
      repo: {
        ...repo.repository,
        latest_commit_status_state: repo.latest_commit_status?.state, // if latest_commit_status is null, it means there is no commit status
        latest_commit_status_state_link: repo.latest_commit_status?.target_url,
        locale_latest_commit_status_state: repo.locale_latest_commit_status,
      },
    })
  }

  let queue = [tree];
  while (queue.length > 0) {
    const item = queue.pop()!;
    queue.push(...item.filter(it => it.kind === "folder").map(it => it.children));
    item.sort((l, r) => {
      if (l.kind === "folder" && r.kind === "folder") { return 0; }
      else if (l.kind === "folder" && r.kind === "repo") { return -1; }
      else if (l.kind === "repo" && r.kind === "folder") { return 1; }
      else if (l.kind === "repo" && r.kind === "repo") { return 0; }
      else { return 0; }
    });
  }

  fullFolderTree.value = tree;
}

function visitSubfolder(name: string) {
  currentFolderPath.value = [...currentFolderPath.value, name];
  updateHistory();
}

function visitParentFolder() {
  currentFolderPath.value = currentFolderPath.value.slice(0, -1);
  updateHistory();
}

onMounted(() => {
  changeReposFilter(reposFilter.value); // the filter dropdown is initialised by the global observer
});

function changeTab(newTab: Tab) {
  tab.value = newTab;
  updateHistory();
}

function changeReposFilter(filter: RepoFilter) {
  reposFilter.value = filter;
  repos.value = [];
  page.value = 1;
  searchRepos();
  loadFolderTree();
}

function updateHistory() {
  const params = new URLSearchParams(window.location.search);

  if (tab.value === 'repos') {
    params.delete('repo-search-tab');
  } else {
    params.set('repo-search-tab', tab.value);
  }

  if (reposFilter.value === 'all') {
    params.delete('repo-search-filter');
  } else {
    params.set('repo-search-filter', reposFilter.value);
  }

  if (privateFilter.value === 'both') {
    params.delete('repo-search-private');
  } else {
    params.set('repo-search-private', privateFilter.value);
  }

  if (archivedFilter.value === 'unarchived') {
    params.delete('repo-search-archived');
  } else {
    params.set('repo-search-archived', archivedFilter.value);
  }

  if (searchQuery.value === '') {
    params.delete('repo-search-query');
  } else {
    params.set('repo-search-query', searchQuery.value);
  }

  if (page.value === 1) {
    params.delete('repo-search-page');
  } else {
    params.set('repo-search-page', `${page.value}`);
  }

  if (currentFolderPath.value.length === 0) {
    params.delete('repo-folder');
  } else {
    params.set('repo-folder', `${JSON.stringify(currentFolderPath.value)}`);
  }

  const queryString = params.toString();
  if (queryString) {
    window.history.replaceState({}, '', `?${queryString}`);
  } else {
    window.history.replaceState({}, '', window.location.pathname);
  }
}

function toggleArchivedFilter() {
  if (archivedFilter.value === 'unarchived') {
    archivedFilter.value = 'archived';
  } else if (archivedFilter.value === 'archived') {
    archivedFilter.value = 'both';
  } else { // including both
    archivedFilter.value = 'unarchived';
  }
  page.value = 1;
  repos.value = [];
  searchRepos();
  loadFolderTree();
}

function togglePrivateFilter() {
  if (privateFilter.value === 'both') {
    privateFilter.value = 'public';
  } else if (privateFilter.value === 'public') {
    privateFilter.value = 'private';
  } else { // including private
    privateFilter.value = 'both';
  }
  page.value = 1;
  repos.value = [];
  searchRepos();
  loadFolderTree();
}

async function changePage(newPage: number) {
  if (isLoading.value) return;

  if (newPage > finalPage.value) newPage = finalPage.value;
  if (newPage < 1) newPage = 1;
  page.value = newPage;
  repos.value = [];
  await searchRepos();
}

async function searchRepos() {
  isLoading.value = true;

  const searchedMode = searchMode.value;
  const searchedURL = searchURL.value;
  const searchedQuery = searchQuery.value;

  let response: Response, json: {data: WebSearchRepo[]};
  try {
    const firstLoad = reposTotalCount.value === null;
    // independent of the search, so both requests go out together
    const totalCountSearchURL = `${subUrl}/repo/search?count_only=1&uid=${uid}&team_id=${teamId}&q=&page=1&mode=`;
    const totalCountRequest = reposTotalCount.value ? null : GET(totalCountSearchURL);
    const searchRequest = GET(searchedURL);
    searchRequest.catch(() => {}); // awaited below, marked handled in case the count throws first
    if (totalCountRequest) {
      reposTotalCount.value = parseInt((await totalCountRequest).headers.get('X-Total-Count') ?? '0');
    }
    if (firstLoad && reposTotalCount.value) {
      nextTick(() => {
        // MDN: If there's no focused element, this is the Document.body or Document.documentElement.
        if ((document.activeElement === document.body || document.activeElement === document.documentElement)) {
          elSearch.value.focus({preventScroll: true});
        }
      });
    }
    response = await searchRequest;
    json = await response.json();
  } catch {
    if (searchedURL === searchURL.value) {
      isLoading.value = false;
      initialSearchDone.value = true;
    }
    return;
  }

  if (searchedURL === searchURL.value) {
    repos.value = json.data.map((webSearchRepo) => {
      return {
        ...webSearchRepo.repository,
        latest_commit_status_state: webSearchRepo.latest_commit_status?.State, // if latest_commit_status is null, it means there is no commit status
        latest_commit_status_state_link: webSearchRepo.latest_commit_status?.TargetURL,
        locale_latest_commit_status_state: webSearchRepo.locale_latest_commit_status,
      };
    });
    const count = parseInt(response.headers.get('X-Total-Count') ?? '0');
    if (searchedQuery === '' && searchedMode === '' && archivedFilter.value === 'both') {
      reposTotalCount.value = count;
    }
    counts.value = {...counts.value, [countsKey.value]: count};
    finalPage.value = Math.ceil(count / searchLimit);
    updateHistory();
    isLoading.value = false;
    initialSearchDone.value = true;
  }
}

function repoIcon(repo: DashboardRepo): SvgName {
  if (repo.fork) {
    return 'octicon-repo-forked';
  } else if (repo.mirror) {
    return 'octicon-mirror';
  } else if (repo.template) {
    return 'octicon-repo-template';
  } else if (repo.private) {
    return 'octicon-lock';
  }
  return 'octicon-repo';
}

function statusIcon(status: CommitStatus) {
  return commitStatus[status].name;
}

function statusColor(status: CommitStatus) {
  return commitStatus[status].color;
}

async function reposFilterKeyControl(e: KeyboardEvent) {
  if (e.isComposing) return;
  switch (e.key) {
    case 'Enter':
      document.querySelector<HTMLAnchorElement>('.repo-owner-name-list li.active a')?.click();
      break;
    case 'ArrowUp':
      if (activeIndex.value > 0) {
        activeIndex.value--;
      } else if (page.value > 1) {
        await changePage(page.value - 1);
        activeIndex.value = searchLimit - 1;
      }
      break;
    case 'ArrowDown':
      if (activeIndex.value < repos.value.length - 1) {
        activeIndex.value++;
      } else if (page.value < finalPage.value) {
        activeIndex.value = 0;
        await changePage(page.value + 1);
      }
      break;
    case 'ArrowRight':
      if (page.value < finalPage.value) {
        await changePage(page.value + 1);
      }
      break;
    case 'ArrowLeft':
      if (page.value > 1) {
        await changePage(page.value - 1);
      }
      break;
  }
  if (activeIndex.value === -1 || activeIndex.value > repos.value.length - 1) {
    activeIndex.value = 0;
  }
}
</script>
<template>
  <div>
    <div v-if="!isOrganization" class="ui three item menu">
      <a :class="{item: true, active: tab === 'repos'}" @click="changeTab('repos')">{{ textRepository }}</a>
      <a :class="{item: true, active: tab === 'folders'}" @click="changeTab('folders')">Folders</a>
      <a :class="{item: true, active: tab === 'organizations'}" @click="changeTab('organizations')">{{ textOrganization }}</a>
    </div>
    <div v-show="tab === 'repos'" class="ui tab active list dashboard-repos">
      <h4 class="ui top attached header tw-flex tw-items-center">
        <div class="tw-flex-1 tw-flex tw-items-center">
          {{ textMyRepos }}
          <span v-if="reposTotalCount" class="ui grey label tw-ml-2">{{ reposTotalCount }}</span>
        </div>
        <a class="tw-flex tw-items-center muted" :href="subUrl + '/repo/create' + (isOrganization ? '?org=' + organizationId : '')" :data-tooltip-content="textNewRepo">
          <svg-icon name="octicon-plus"/>
        </a>
      </h4>
      <div v-if="!reposTotalCount" class="ui attached segment">
        <div v-if="!isLoading" class="empty-repo-or-org">
          <svg-icon name="octicon-git-branch" :size="24"/>
          <p>{{ textNoRepo }}</p>
        </div>
        <!-- using the loading indicator here will cause more (unnecessary) page flickers, so at the moment, not use the loading indicator -->
        <!-- <div v-else class="is-loading loading-icon-2px tw-min-h-16"/> -->
      </div>
      <div v-else class="ui attached segment repos-search">
        <div class="ui small fluid action left icon input">
          <input type="search" spellcheck="false" maxlength="255" @input="changeReposFilter(reposFilter)" v-model="searchQuery" ref="elSearch" @keydown="reposFilterKeyControl" :placeholder="textSearchRepos">
          <i class="icon loading-icon-3px" :class="{'is-loading': isLoading}"><svg-icon name="octicon-search" :size="16"/></i>
          <div class="ui dropdown icon button" :title="textFilter">
            <svg-icon name="octicon-filter" :size="16"/>
            <div class="menu">
              <a class="item" @click="toggleArchivedFilter()">
                <div class="ui checkbox" :title="checkboxArchivedFilterTitle">
                  <!--the "tw-pointer-events-none" is necessary to prevent the checkbox from handling user's input,
                      otherwise if the "input" handles click event for intermediate status, it breaks the internal state-->
                  <input type="checkbox" class="tw-pointer-events-none" v-bind.prop="checkboxArchivedFilterProps">
                  <label>
                    <svg-icon name="octicon-archive" :size="16" class="tw-mr-1"/>
                    {{ textShowArchived }}
                  </label>
                </div>
              </a>
              <a class="item" @click="togglePrivateFilter()">
                <div class="ui checkbox" :title="checkboxPrivateFilterTitle">
                  <input type="checkbox" class="tw-pointer-events-none" v-bind.prop="checkboxPrivateFilterProps">
                  <label>
                    <svg-icon name="octicon-lock" :size="16" class="tw-mr-1"/>
                    {{ textShowPrivate }}
                  </label>
                </div>
              </a>
            </div>
          </div>
        </div>
        <!-- stay hidden until the first count arrives, otherwise the label resizes after paint -->
        <overflow-menu class="ui secondary pointing tabular borderless menu repos-filter" :class="{'tw-invisible': !initialSearchDone}">
          <div class="overflow-menu-items tw-justify-center">
            <a class="item" tabindex="0" :class="{active: reposFilter === 'all'}" @click="changeReposFilter('all')">
              {{ textAll }}
              <div v-show="reposFilter === 'all'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'sources'}" @click="changeReposFilter('sources')">
              {{ textSources }}
              <div v-show="reposFilter === 'sources'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'forks'}" @click="changeReposFilter('forks')">
              {{ textForks }}
              <div v-show="reposFilter === 'forks'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'mirrors'}" @click="changeReposFilter('mirrors')" v-if="isMirrorsEnabled">
              {{ textMirrors }}
              <div v-show="reposFilter === 'mirrors'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'collaborative'}" @click="changeReposFilter('collaborative')">
              {{ textCollaborative }}
              <div v-show="reposFilter === 'collaborative'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
          </div>
        </overflow-menu>
      </div>
      <div v-if="repos.length" class="ui attached table segment tw-rounded-b">
        <ul class="repo-owner-name-list">
          <li class="tw-flex tw-items-center tw-py-2" v-for="(repo, index) in repos" :class="{'active': index === activeIndex}" :key="repo.id">
            <a class="repo-list-link muted" :href="repo.link">
              <svg-icon :name="repoIcon(repo)" :size="16" class="repo-list-icon"/>
              <div class="tw-inline-block tw-truncate">{{ repo.full_name }}</div>
              <div v-if="repo.archived">
                <svg-icon name="octicon-archive" :size="16"/>
              </div>
            </a>
            <a class="tw-flex tw-items-center" v-if="repo.latest_commit_status_state" :href="repo.latest_commit_status_state_link || undefined" :data-tooltip-content="repo.locale_latest_commit_status_state">
              <!-- the commit status icon logic is taken from templates/repo/commit_status.tmpl -->
              <svg-icon :name="statusIcon(repo.latest_commit_status_state)" :class="'tw-ml-2 commit-status icon ' + statusColor(repo.latest_commit_status_state)" :size="16"/>
            </a>
          </li>
        </ul>
        <div v-if="showMoreReposLink" class="tw-text-center">
          <div class="divider tw-my-0"/>
          <div class="ui borderless pagination menu narrow tw-my-2">
            <a
              class="item navigation tw-py-1" :class="{'disabled': page === 1}"
              @click="changePage(1)" :title="textFirstPage"
            >
              <svg-icon name="gitea-double-chevron-left" :size="16" class="tw-mr-1"/>
            </a>
            <a
              class="item navigation tw-py-1" :class="{'disabled': page === 1}"
              @click="changePage(page - 1)" :title="textPreviousPage"
            >
              <svg-icon name="octicon-chevron-left" :size="16" class="tw-mr-1"/>
            </a>
            <a class="active item tw-py-1">{{ page }}</a>
            <a
              class="item navigation" :class="{'disabled': page === finalPage}"
              @click="changePage(page + 1)" :title="textNextPage"
            >
              <svg-icon name="octicon-chevron-right" :size="16" class="tw-ml-1"/>
            </a>
            <a
              class="item navigation tw-py-1" :class="{'disabled': page === finalPage}"
              @click="changePage(finalPage)" :title="textLastPage"
            >
              <svg-icon name="gitea-double-chevron-right" :size="16" class="tw-ml-1"/>
            </a>
          </div>
        </div>
      </div>
    </div>
    <div v-if="!isOrganization" v-show="tab === 'organizations'" class="ui tab active list dashboard-orgs">
      <h4 class="ui top attached header tw-flex tw-items-center">
        <div class="tw-flex-1 tw-flex tw-items-center">
          {{ textMyOrgs }}
          <span class="ui grey label tw-ml-2">{{ organizationsTotalCount }}</span>
        </div>
        <a class="tw-flex tw-items-center muted" v-if="canCreateOrganization" :href="subUrl + '/org/create'" :data-tooltip-content="textNewOrg">
          <svg-icon name="octicon-plus"/>
        </a>
      </h4>
      <div v-if="!organizations.length" class="ui attached segment">
        <div class="empty-repo-or-org">
          <svg-icon name="octicon-organization" :size="24"/>
          <p>{{ textNoOrg }}</p>
        </div>
      </div>
      <div v-else class="ui attached table segment tw-rounded-b">
        <ul class="repo-owner-name-list">
          <li class="tw-flex tw-items-center tw-py-2" v-for="org in organizations" :key="org.name">
            <a class="repo-list-link muted" :href="subUrl + '/' + encodeURIComponent(org.name)">
              <svg-icon name="octicon-organization" :size="16" class="repo-list-icon"/>
              <div class="tw-inline-block tw-truncate">{{ org.full_name ? `${org.full_name} (${org.name})` : org.name }}</div>
              <div><!-- div to prevent underline of label on hover -->
                <span class="ui tiny basic label" v-if="org.org_visibility !== 'public'">
                  {{ org.org_visibility === 'limited' ? textOrgVisibilityLimited: textOrgVisibilityPrivate }}
                </span>
              </div>
            </a>
            <div class="tw-text-grey-light tw-flex tw-items-center tw-ml-2">
              {{ org.num_repos }}
              <svg-icon name="octicon-repo" :size="16" class="tw-ml-1 tw-mt-0.5"/>
            </div>
          </li>
        </ul>
      </div>
    </div>
    <div v-if="!isOrganization" v-show="tab === 'folders'" class="ui tab active list dashboard-orgs">
      <h4 class="ui top attached header tw-flex tw-items-center">
        <div class="tw-flex-1 tw-flex tw-items-center">
          {{ textMyRepos }}
          <span v-if="reposTotalCount" class="ui grey label tw-ml-2">{{ reposTotalCount }}</span>
        </div>
        <a class="tw-flex tw-items-center muted" :href="subUrl + '/repo/create' + (isOrganization ? '?org=' + organizationId : '')" :data-tooltip-content="textNewRepo">
          <svg-icon name="octicon-plus"/>
        </a>
      </h4>
      <div v-if="!reposTotalCount" class="ui attached segment">
        <div v-if="!isLoading" class="empty-repo-or-org">
          <svg-icon name="octicon-git-branch" :size="24"/>
          <p>{{ textNoRepo }}</p>
        </div>
        <!-- using the loading indicator here will cause more (unnecessary) page flickers, so at the moment, not use the loading indicator -->
        <!-- <div v-else class="is-loading loading-icon-2px tw-min-h-16"/> -->
      </div>
      <div v-else class="ui attached segment repos-search">
        <div class="ui small fluid action left icon input">
          <input type="search" spellcheck="false" maxlength="255" @input="changeReposFilter(reposFilter)" v-model="searchQuery" ref="elSearch" @keydown="reposFilterKeyControl" :placeholder="textSearchRepos">
          <i class="icon loading-icon-3px" :class="{'is-loading': isLoading}"><svg-icon name="octicon-search" :size="16"/></i>
          <div class="ui dropdown icon button" :title="textFilter">
            <svg-icon name="octicon-filter" :size="16"/>
            <div class="menu">
              <a class="item" @click="toggleArchivedFilter()">
                <div class="ui checkbox" :title="checkboxArchivedFilterTitle">
                  <!--the "tw-pointer-events-none" is necessary to prevent the checkbox from handling user's input,
                      otherwise if the "input" handles click event for intermediate status, it breaks the internal state-->
                  <input type="checkbox" class="tw-pointer-events-none" v-bind.prop="checkboxArchivedFilterProps">
                  <label>
                    <svg-icon name="octicon-archive" :size="16" class="tw-mr-1"/>
                    {{ textShowArchived }}
                  </label>
                </div>
              </a>
              <a class="item" @click="togglePrivateFilter()">
                <div class="ui checkbox" :title="checkboxPrivateFilterTitle">
                  <input type="checkbox" class="tw-pointer-events-none" v-bind.prop="checkboxPrivateFilterProps">
                  <label>
                    <svg-icon name="octicon-lock" :size="16" class="tw-mr-1"/>
                    {{ textShowPrivate }}
                  </label>
                </div>
              </a>
            </div>
          </div>
        </div>
        <!-- stay hidden until the first count arrives, otherwise the label resizes after paint -->
        <overflow-menu class="ui secondary pointing tabular borderless menu repos-filter" :class="{'tw-invisible': !initialSearchDone}">
          <div class="overflow-menu-items tw-justify-center">
            <a class="item" tabindex="0" :class="{active: reposFilter === 'all'}" @click="changeReposFilter('all')">
              {{ textAll }}
              <div v-show="reposFilter === 'all'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'sources'}" @click="changeReposFilter('sources')">
              {{ textSources }}
              <div v-show="reposFilter === 'sources'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'forks'}" @click="changeReposFilter('forks')">
              {{ textForks }}
              <div v-show="reposFilter === 'forks'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'mirrors'}" @click="changeReposFilter('mirrors')" v-if="isMirrorsEnabled">
              {{ textMirrors }}
              <div v-show="reposFilter === 'mirrors'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
            <a class="item" tabindex="0" :class="{active: reposFilter === 'collaborative'}" @click="changeReposFilter('collaborative')">
              {{ textCollaborative }}
              <div v-show="reposFilter === 'collaborative'" class="ui circular mini grey label">{{ repoTypeCount }}</div>
            </a>
          </div>
        </overflow-menu>
      </div>
      <div v-if="currentFolder.length" class="ui attached table segment tw-rounded-b">
        <ul class="repo-owner-name-list">
          <li v-if="currentFolderPath.length" class="tw-flex tw-items-center tw-py-2">
            <a class="repo-list-link muted" @click="visitParentFolder()">
              <svg-icon :name="'octicon-chevron-up'" :size="16" class="repo-list-icon"/>
              <div class="tw-inline-block tw-truncate">{{ currentFolderPath.join("/") }}</div>
            </a>
          </li>

          <li class="tw-flex tw-items-center tw-py-2" v-for="item in currentFolder" :key="item.name">
            <a v-if="item.kind === 'repo'" class="repo-list-link muted" :href="item.repo.link">
              <svg-icon :name="repoIcon(item.repo)" :size="16" class="repo-list-icon"/>
              <div class="tw-inline-block tw-truncate">{{ item.name }}</div>
              <div v-if="item.repo.archived">
                <svg-icon name="octicon-archive" :size="16"/>
              </div>
            </a>
            <a v-if="item.kind === 'repo' && item.repo.latest_commit_status_state" class="tw-flex tw-items-center" :href="item.repo.latest_commit_status_state_link || undefined" :data-tooltip-content="item.repo.locale_latest_commit_status_state">
              <!-- the commit status icon logic is taken from templates/repo/commit_status.tmpl -->
              <svg-icon :name="statusIcon(item.repo.latest_commit_status_state)" :class="'tw-ml-2 commit-status icon ' + statusColor(item.repo.latest_commit_status_state)" :size="16"/>
            </a>

            <a v-if="item.kind === 'folder'" class="repo-list-link muted" @click="visitSubfolder(item.name)">
              <img v-if="!currentFolderPath.length" loading="lazy" class="ui avatar tw-align-middle" height="16" width="16" style="margin-right: 2px;" :src="subUrl + '/avatars/' + (avatars[item.name] ?? '') + '?size=48'" alt="">
              <svg-icon v-if="currentFolderPath.length" :name="'octicon-file-directory-fill'" :size="16" class="repo-list-icon"/>
              <div class="tw-inline-block tw-truncate">{{ item.name }}</div>
            </a>
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>
<style scoped>
ul {
  list-style: none;
  margin: 0;
  padding-left: 0;
}

ul li {
  padding: 0 10px;
}

ul li:not(:last-child) {
  border-bottom: 1px solid var(--color-secondary);
}

.repos-search {
  padding-bottom: 0 !important;
}

.repos-filter {
  margin-top: 0 !important;
  border-bottom-width: 0 !important;
}

.repos-filter .item {
  padding-left: 6px !important;
  padding-right: 6px !important;
}

.repo-list-link {
  min-width: 0; /* for text truncation */
  display: flex;
  align-items: center;
  flex: 1;
  gap: 0.5rem;
}

.repo-list-link .svg {
  color: var(--color-text-light-2);
}

.repo-list-icon {
  min-width: 16px;
  margin-right: 2px;
}

/* octicon-mirror has no padding inside the SVG */
.repo-list-icon.octicon-mirror {
  width: 14px;
  min-width: 14px;
  margin-left: 1px;
  margin-right: 3px;
}

.repo-owner-name-list li.active {
  background: var(--color-hover);
}

.empty-repo-or-org {
  margin-top: 1em;
  text-align: center;
  color: var(--color-placeholder-text);
}

.empty-repo-or-org p {
  margin: 1em auto;
}
</style>
