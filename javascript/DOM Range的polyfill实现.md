在实现，例如富文本编辑器这样的需求时，我们通常都需要自己实现一套 DOM Range 以弥补IE的问题，以下是我根据Whatwg对 DOM Range 的定义[文档](https://dom.spec.whatwg.org/#ranges)，实现的一套 DOM Range polyfill

```typescript
module RangePolyfill {

const enum NodeType {
    ELEMENT_NODE = 1,
    ATTRIBUTE_NODE,
    TEXT_NODE,
    CDATA_SECTION_NODE,
    ENTITY_REFERENCE_NODE,
    ENTITY_NODE,
    PROCESSING_INSTRUCTION_NODE,
    COMMENT_NODE,
    DOCUMENT_NODE,
    DOCUMENT_TYPE_NODE,
    DOCUMENT_FRAGMENT_NODE,
    NOTATION_NODE
};

const enum Position {
    NONE,
    BEFORE,
    EQUAL,
    AFTER
};

export class RangeDOMException extends DOMException {
    constructor(public message: string, public name: string) {
        super();
    }
}

export class Range {
    private rootContainer: Document

    collapsed: boolean
    commonAncestorContainer: Node
    startContainer: Node
    endContainer: Node
    startOffset: number
    endOffset: number

    constructor() {
        this.collapsed = true;
        this.rootContainer =
        this.commonAncestorContainer =
        this.startContainer =
        this.endContainer = document;
        this.startOffset =
        this.endOffset = 0;
    }
    
    private isText(node: Node): node is Text {
        return node.nodeType === NodeType.TEXT_NODE;
    }

    private isElement(node: Node): node is Element {
        return node.nodeType === NodeType.ELEMENT_NODE;
    }

    private isTextData(node: Node): node is Text {
        return node.nodeType === NodeType.COMMENT_NODE || node.nodeType === NodeType.CDATA_SECTION_NODE;
    }

    private isGeneralizedText(node: Node): node is Text {
        return this.isText(node) || this.isTextData(node);
    }

    private getAncestorList(node: Node, until?: Node): Array<Node> {
        const nodeList = [];

        while (node && (!until || until !== node)) {
            nodeList.unshift(node);
            node = node.parentNode;
        }

        return nodeList;
    }

    private getNodeOffset(node: Node): number {
        let offset = 0;
        let each = node.previousSibling
        
        while (each) {
            offset++;
            each = each.previousSibling;
        }

        return offset;
    }

    private getNodeLength(node: Node): number {
        if (this.isGeneralizedText(node)) {
            return node.nodeValue.length;
        } else {
            return node.childNodes.length;
        }
    }

    private getTreeCount(node: Node): number {
        let count = 0;
        let each = node.firstChild;

        while (each) {
            count = this.getTreeCount(each);
            each = each.nextSibling;
        }

        return count + 1;
    }

    private getTreeOrder(node: Node): number {
        if (node.parentNode === null) {
            return 0;
        }

        const parentOrder = this.getTreeOrder(node.parentNode);

        let count = 0;
        let each = node.previousSibling;

        while (each) {
            count += this.getTreeCount(each);
            each = each.previousSibling;
        }

        return parentOrder + count + 1;
    }

    private eachChild(node: Node, callback: (node: Node) => void): void {
        let each = node.firstChild;

        while (each) {
            const next = each.nextSibling;

            callback(each);

            each = next;
        };
    }

    private comparePosition(startNode, startOffset, endNode, endOffset): Position {
        if (startNode === endNode) {
            // 如果两个节点为同一节点，则比较offset值
            if (startOffset < endOffset) {
                return Position.BEFORE;
            } else if (startOffset > endOffset) {
                return Position.AFTER;
            } else {
                return Position.EQUAL;
            }
        } else if (startNode.contains(endNode)) {
            // 如果startNode是endNode的祖先节点
            const refNode = this.getAncestorList(endNode, startNode)[0];
            const refOffset = this.getNodeOffset(refNode);

            if (startOffset > refOffset) {
                return Position.AFTER;
            } else {
                return Position.BEFORE;
            }
        } else if (endNode.contains(startNode)) {
            // 如果endNode是startNode的祖先节点
            const refNode = this.getAncestorList(startNode, endNode)[0];
            const refOffset = this.getNodeOffset(refNode);

            if (endOffset > refOffset) {
                return Position.BEFORE;
            } else {
                return Position.AFTER;
            }
        } else {
            const startOrder = this.getTreeOrder(startNode);
            const endOrder = this.getTreeOrder(endNode);

            if (startOrder > endOrder) {
                return Position.AFTER;
            } else {
                return Position.BEFORE;
            }
        }
    }
    
    private getCommonAncestor(): Node | null {
        if (this.startContainer === this.endContainer) {
            return this.startContainer;
        }

        const startAncestorList = this.getAncestorList(this.startContainer);
        const endAncestorList = this.getAncestorList(this.endContainer);

        let i = 0;

        while (startAncestorList[i] === endAncestorList[i]) {
            i++;
        }

        return i === 0 ? null : startAncestorList[i - 1];
    }
    
    private isCollapsed() {
        return this.startContainer === this.endContainer && this.startOffset === this.endOffset;
    }

    setStart(node: Node, offset: number): void {
        if (offset > this.getNodeLength(node)) {
            throw new RangeDOMException(`Failed to execute 'setStart' on 'Range': The offset ${ offset } is larger than the node's length (${ this.getNodeLength(node) }).`, "IndexSizeError")
        }

        if (!this.rootContainer.contains(node) ||
            this.comparePosition(node, offset, this.endContainer, this.endOffset) === Position.AFTER) {
            this.endContainer = node;
            this.endOffset = offset;
        }

        this.startContainer = node;
        this.startOffset = offset;
        this.rootContainer = node.ownerDocument || (node as Document);

        this.collapsed = this.isCollapsed();
        this.commonAncestorContainer = this.getCommonAncestor();
    }

    setEnd(node: Node, offset: number): void {
        if (offset > this.getNodeLength(node)) {
            throw new RangeDOMException(`Failed to execute 'setEnd' on 'Range': The offset ${ offset } is larger than the node's length (${ this.getNodeLength(node) }).`, "IndexSizeError")
        }

        if (!this.rootContainer.contains(node) ||
            this.comparePosition(node, offset, this.startContainer, this.startOffset) === Position.BEFORE) {
            this.startContainer = node;
            this.startOffset = offset;
        }

        this.endContainer = node;
        this.endOffset = offset;
        this.rootContainer = node.ownerDocument || (node as Document);
        
        this.collapsed = this.isCollapsed();
        this.commonAncestorContainer = this.getCommonAncestor();
    }

    setStartBefore(node: Node): void {
        if (!node.parentNode) {
            throw new RangeDOMException("Failed to execute 'setStartBefore' on 'Range': the given Node has no parent.", "InvalidNodeTypeError");
        }

        this.setStart(node.parentNode, this.getNodeOffset(node));
    }

    setStartAfter(node: Node): void {
        if (!node.parentNode) {
            throw new RangeDOMException("Failed to execute 'setStartAfter' on 'Range': the given Node has no parent.", "InvalidNodeTypeError");
        }

        this.setStart(node.parentNode, this.getNodeOffset(node) + 1);
    }

    setEndBefore(node: Node): void {
        if (!node.parentNode) {
            throw new RangeDOMException("Failed to execute 'setEndBefore' on 'Range': the given Node has no parent.", "InvalidNodeTypeError");
        }

        this.setEnd(node.parentNode, this.getNodeOffset(node));
    }

    setEndAfter(node: Node): void {
        if (!node.parentNode) {
            throw new RangeDOMException("Failed to execute 'setEndAfter' on 'Range': the given Node has no parent.", "InvalidNodeTypeError");
        }

        this.setEnd(node.parentNode, this.getNodeOffset(node) + 1);
    }

    collapse(toStart: boolean = true): void {
        if (toStart) {
            this.endContainer = this.startContainer;
            this.endOffset = this.startOffset;
        } else {
            this.startContainer = this.endContainer;
            this.startOffset = this.endOffset;
        }

        this.collapsed = true;
    }

    selectNode(node: Node): void {
        if (!node.parentNode) {
            throw new RangeDOMException("Failed to execute 'selectNode' on 'Range': the given Node has no parent.", "InvalidNodeTypeError");
        }

        this.setStartBefore(node);
        this.setEndAfter(node);
    }

    selectNodeContents(node: Node): void {
        this.setStart(node, 0);
        this.setEnd(node, this.getNodeLength(node));
    }

    private extractText(node: Text, offset: number, count: number, isClone = false): Node | null {
        let clone = node.cloneNode(false) as Text;
        clone.data = node.substringData(offset, count);

        if (!isClone) {
            node.replaceData(offset, count, "");
        }

        return clone;
    }

    private extractPartially(startNode: Node, startOffset: number, endNode: Node, endOffset: number, isClone = false): Node | null {
        if (startNode === endNode && this.isGeneralizedText(startNode)) {
            return this.extractText(startNode, startOffset, endOffset - startOffset, isClone);
        }
        
        let clone: Node = null;

        if (startNode.contains(endNode)) {
            clone = startNode.cloneNode(false);
        } else {
            clone = endNode.cloneNode(false);
        }

        const range = new Range();
        range.setStart(startNode, startOffset);
        range.setEnd(endNode, endOffset);

        const fragment = range.extractInAction(isClone);
        clone.appendChild(fragment);

        return clone;
    }

    private extractInAction(isClone = false): DocumentFragment {
        const fragment = this.rootContainer.createDocumentFragment();

        if (this.collapsed) {
            return fragment;
        }
        
        const startNode = this.startContainer;
        const startOffset = this.startOffset;
        const endNode = this.endContainer;
        const endOffset = this.endOffset;

        if (startNode === endNode && this.isGeneralizedText(startNode)) {
            fragment.appendChild(this.extractText(startNode, startOffset, endOffset - startOffset, isClone));
            return fragment;
        }

        let firstPartiallyContainedChild: Node = null;

        if (!startNode.contains(endNode)) {
            firstPartiallyContainedChild = this.getAncestorList(startNode, this.commonAncestorContainer)[0];
        }

        let lastPartiallyContainedChild: Node = null;

        if (!endNode.contains(startNode)) {
            lastPartiallyContainedChild = this.getAncestorList(endNode, this.commonAncestorContainer)[0];
        }

        const containedChildren = [] as Array<Node>;

        this.eachChild(this.commonAncestorContainer, (each) => {
            if (this.comparePosition(each, 0, startNode, startOffset) === Position.AFTER &&
                this.comparePosition(each, this.getNodeLength(each), endNode, endOffset) === Position.BEFORE) {
                containedChildren.push(each);
            }
        });

        let newNode: Node;
        let newOffset: number;

        if (startNode.contains(endNode)) {
            newNode = startNode;
            newOffset = startOffset;
        } else {
            const refNode = this.getAncestorList(startNode, this.commonAncestorContainer)[0];

            newNode = refNode.parentNode;
            newOffset = this.getNodeOffset(refNode) + 1;
        }

        if (firstPartiallyContainedChild !== null) {
            fragment.appendChild(
                this.extractPartially(startNode, startOffset, firstPartiallyContainedChild, this.getNodeLength(firstPartiallyContainedChild), isClone)
            );
        }
        
        for (let i = 0; i < containedChildren.length; i++) {
            const node = containedChildren[i];
            fragment.appendChild(isClone ? node.cloneNode(true) : node);
        }

        if (lastPartiallyContainedChild !== null) {
            fragment.appendChild(
                this.extractPartially(lastPartiallyContainedChild, 0, endNode, endOffset, isClone)
            );
        }
        
        if (!isClone) {
            this.setStart(newNode, newOffset);
            this.setEnd(newNode, newOffset);
        }

        return fragment;
    }
    
    deleteContents(): void {
        this.extractInAction();
    }

    extractContents(): DocumentFragment {
        return this.extractInAction(false);
    }
    
    cloneContents(): DocumentFragment {
        return this.extractInAction(true);
    }
    
    insertNode(node: Node) {
        if (this.isTextData(node)) {
            throw new RangeDOMException(`Failed to execute 'insertNode' on 'Range': The node to be inserted is a ${ node.nodeName } node, which may not be inserted here.`, "HierarchyRequestError");
        }

        if (this.isText(node) && !node.parentNode) {
            throw new RangeDOMException("Failed to execute 'insertNode' on 'Range': This operation would split a text node, but there's no parent into which to insert.", "HierarchyRequestError");
        }

        const startNode = this.startContainer;
        const startOffset = this.startOffset;

        if (this.isText(startNode)) {
            const parentNode = startNode.parentNode;

            if (startOffset === 0) {
                parentNode.insertBefore(node, startNode);
            } else if (startOffset >= startNode.nodeValue.length) {
                const nextSibling = startNode.nextSibling;
                
                if (nextSibling) {
                    parentNode.insertBefore(node, nextSibling);
                } else {
                    parentNode.appendChild(node);
                }
            } else {
                const nextSibling = startNode.splitText(startOffset);

                parentNode.insertBefore(node, nextSibling);
            }
        } else {
            const nextSibling = startNode.childNodes[startOffset];

            if (nextSibling) {
                startNode.insertBefore(node, nextSibling);
            } else {
                startNode.appendChild(node);
            }
        }
    }

    surroundContents(newParent: Node): void {
        let startNode = this.startContainer;
        let endNode = this.endContainer;

        if (this.isGeneralizedText(startNode)) startNode = startNode.parentNode;
        if (this.isGeneralizedText(endNode)) endNode = endNode.parentNode;

        if (startNode !== endNode) {
            throw new RangeDOMException("Failed to execute 'surroundContents' on 'Range': The Range has partially selected a non-Text node.", "InvalidStateError");
        }

        let fragment = this.extractContents();

        while (newParent.childNodes.length) {
            newParent.removeChild(newParent.firstChild);
        }

        this.insertNode(newParent);

        newParent.appendChild(fragment);

        this.selectNode(newParent);
    }

    cloneRange() {
        const clone = new Range();

        clone.setStart(this.startContainer, this.startOffset);
        clone.setEnd(this.endContainer, this.endOffset);

        return clone;
    }

    isPointInRange(node: Node, offset: number): boolean {
        if (offset > this.getNodeLength(node)) {
            throw new RangeDOMException(`Failed to execute 'isPointInRange' on 'Range': There is no child at offset ${ offset }.`, "IndexSizeError");
        }

        if (!this.rootContainer.contains(node)) {
            return false;
        }

        if (this.comparePosition(node, offset, this.startContainer, this.startOffset) === Position.BEFORE ||
            this.comparePosition(node, offset, this.endContainer, this.endOffset) === Position.AFTER) {
            return false;
        }

        return true;
    }

    comparePoint(node: Node, offset: number) {
        if (!this.rootContainer.contains(node)) {
            throw new RangeDOMException("Failed to execute 'comparePoint' on 'Range': The node provided and the Range are not in the same tree.", "WrongDocumentError");
        }

        if (offset > this.getNodeLength(node)) {
            throw new RangeDOMException(`Failed to execute 'isPointInRange' on 'Range': There is no child at offset ${ offset }.`, "IndexSizeError");
        }

        if (this.comparePosition(node, offset, this.startContainer, this.startOffset) === Position.BEFORE) {
            return -1;
        }

        if (this.comparePosition(node, offset, this.endContainer, this.endOffset) === Position.AFTER) {
            return 1;
        }

        return 0;
    }

    intersectsNode(node: Node): boolean {
        if (!this.rootContainer.contains(node)) {
            return false;
        }

        const parent = node.parentNode;

        if (parent === null) {
            return true;
        }

        const offset = this.getNodeOffset(node);

        if (this.comparePosition(parent, offset, this.endContainer, this.endOffset) === Position.BEFORE &&
            this.comparePosition(parent, offset + 1, this.startContainer, this.startOffset) === Position.AFTER) {
            return true;
        }

        return false;
    }
}

}
```