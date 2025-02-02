<template>
  <div class="blog-post-list-container">
    <!-- Debug Format -->
    <div v-if="format === 'debug'">
      <pre>{{ JSON.stringify(displayedPosts, null, 2) }}</pre>
    </div>

    <!-- Dynamic Card Rendering -->
    <component :is="containerComponent" class="cards-container">
      <TitleCard
        v-if="showTitleCard"
        :title="headerTitle"
        :date="headerDate"
        :title-lines="headerTitleLines"
        :link="headerLink"
      />
      <component
        v-for="post in displayedPosts"
        :key="post.url"
        :is="selectedCardComponent"
        v-bind="getCardProps(post)"
      />
    </component>
  </div>
</template>

<script setup lang="ts">
import { inject, computed } from "vue";
import { formatDate } from "@/utils/MyUtils";
import HorizontalCard from "./cards/HorizontalCard.vue";
import VerticalCard from "./cards/VerticalCard.vue";
import HorizontalContainer from "./containers/HorizontalContainer.vue";
import VerticalContainer from "./containers/VerticalContainer.vue";
import TitleCard from "./cards/HeaderCard.vue";

interface Frontmatter {
  title: string;
  excerpt: string;
  date: string;
  banner?: string;
  category?: string;
  author?: string;
  featured?: boolean;
  draft?: boolean;
}

interface Post {
  url: string;
  frontmatter: Frontmatter;
}

interface Author {
  name: string;
  [key: string]: any;
}

interface BlogPostListProps {
  headerTitle?: string;
  headerTitleLines?: number;
  headerLink?: string;
  headerDate?: string;
  posts?: Post[];
  format?: "debug" | "vertical" | "horizontal";
  sortOrder?: "ascending" | "descending";
  startDate?: Date | string | null;
  endDate?: Date | string | null;
  renderDrafts?: boolean;
  featuredOnly?: boolean;
  filterAuthors?: string[];
  excludeAuthors?: string[];
  filterCategories?: string[];
  excludeCategories?: string[];
  excludeURLs?: string[];
  maxCards?: number | null;
  hideAuthor?: boolean;
  hideDate?: boolean;
  hideImage?: boolean;
  hideCategory?: boolean;
  hideDomain?: boolean;
  disableLinks?: boolean;
  titleLines?: number;
  excerptLines?: number;
  postsDataKey?: string;
  authorsDataKey?: string;
}

const props = withDefaults(defineProps<BlogPostListProps>(), {
  headerTitle: "",
  headerDate: "",
});

const showTitleCard = computed(() => {
  return !!(props.headerTitle || props.headerDate || props.headerLink);
});

const injectedPostsData = inject<Post[]>(props.postsDataKey || "postsData", []);
const authors = inject<Record<string, Author>>(props.authorsDataKey || "authors", {});

const posts = computed(() => props.posts || injectedPostsData);

// Card & Container selection
const selectedCardComponent = computed(() => {
  switch (props.format) {
    case "horizontal":
      return HorizontalCard;
    case "vertical":
      return VerticalCard;
    default:
      return VerticalCard;
  }
});

const containerComponent = computed(() => {
  switch (props.format) {
    case "horizontal":
      return HorizontalContainer;
    case "vertical":
      return VerticalContainer;
    default:
      return VerticalContainer;
  }
});

const sortedPosts = computed(() => {
  const sorted = [...posts.value];
  sorted.sort((a, b) => {
    const dateA = new Date(a.frontmatter.date).getTime();
    const dateB = new Date(b.frontmatter.date).getTime();
    return props.sortOrder === "ascending" ? dateA - dateB : dateB - dateA;
  });
  return sorted;
});

const filteredPosts = computed(() => {
  return sortedPosts.value.filter((post) => {
    const { frontmatter } = post;

    if (props.featuredOnly && !frontmatter.featured) return false;
    if (!props.renderDrafts && frontmatter.draft) return false;

    const postDate = new Date(frontmatter.date).getTime();
    if (props.startDate && postDate < new Date(props.startDate).getTime()) return false;
    if (props.endDate && postDate > new Date(props.endDate).getTime()) return false;

    if (
      props.filterAuthors?.length &&
      !props.filterAuthors.includes(frontmatter.author || "")
    )
      return false;
    if (
      props.excludeAuthors?.length &&
      props.excludeAuthors.includes(frontmatter.author || "")
    )
      return false;

    if (
      props.filterCategories?.length &&
      !props.filterCategories.includes(frontmatter.category || "")
    )
      return false;
    if (
      props.excludeCategories?.length &&
      props.excludeCategories.includes(frontmatter.category || "")
    )
      return false;

    if (props.excludeURLs?.length) {
      const postURL = post.url.replace(/\.html$/, "");
      const isExcluded = props.excludeURLs.some((excludeURL) => {
        const normalizedExcludeURL = excludeURL.replace(/\.html$/, "");
        return normalizedExcludeURL === postURL;
      });
      if (isExcluded) return false;
    }

    return true;
  });
});

const displayedPosts = computed(() => {
  if (props.maxCards != null && props.maxCards >= 0) {
    return filteredPosts.value.slice(0, props.maxCards);
  }
  return filteredPosts.value;
});

// Utility functions
// function formatDate(date: string | Date): string {
//   return new Date(date).toLocaleDateString(undefined, {
//     year: "numeric",
//     month: "long",
//     day: "numeric",
//   });
// }

function getAuthorName(authorKey: string): string {
  const author = authors[authorKey];
  return author ? author.name : authorKey;
}

function getCardProps(post: Post) {
  return {
    title: post.frontmatter.title,
    excerpt: post.frontmatter.excerpt,
    url: post.url,
    hideDomain: props.hideDomain,
    isExternal: false,
    author: getAuthorName(post.frontmatter.author || ""),
    date: formatDate(post.frontmatter.date),
    image: post.frontmatter.banner,
    category: post.frontmatter.category,
    hideAuthor: props.hideAuthor,
    hideDate: props.hideDate,
    hideImage: props.hideImage,
    hideCategory: props.hideCategory,
    disableLinks: props.disableLinks,
    titleLines: props.titleLines,
    excerptLines: props.excerptLines,
  };
}
</script>

<style scoped>
.blog-post-list-container {
  padding: 1rem 0;
}
</style>
